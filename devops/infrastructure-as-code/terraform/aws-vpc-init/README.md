# Terraform AWS VPC Provisioning with Variable-Driven Configuration

> **Project:** Infrastructure as Code | **Platform:** AWS (LocalStack) | **Tool:** Terraform `v5.91.0 (hashicorp/aws)` | **Environment:** Linux (Ubuntu) | **Scope:** Network Infrastructure

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Inspect the Working Directory](#step-1-inspect-the-working-directory)
  - [Step 2: Create the Variables File](#step-2-create-the-variables-file)
  - [Step 3: Create the Main Configuration File](#step-3-create-the-main-configuration-file)
  - [Step 4: Verify the Working Directory](#step-4-verify-the-working-directory)
  - [Step 5: Initialize Terraform](#step-5-initialize-terraform)
  - [Step 6: Validate the Configuration](#step-6-validate-the-configuration)
  - [Step 7: Plan the Infrastructure Changes](#step-7-plan-the-infrastructure-changes)
  - [Step 8: Apply the Configuration](#step-8-apply-the-infrastructure-changes)
- [Configuration Reference](#configuration-reference)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)

---

## Overview

This project provisions an AWS Virtual Private Cloud (VPC) using **Terraform** with a variable-driven configuration pattern. The implementation separates infrastructure definitions from their configurable values, enforcing modularity and reusability as foundational IaC principles.

The VPC is created against a **LocalStack** endpoint (simulated AWS at `http://aws:4566`), making this approach suitable for local development, CI/CD pipeline testing, and sandbox environments without incurring AWS costs or requiring live credentials.

---

## Problem Statement

The Nautilus DevOps team requires automated, reproducible VPC provisioning as part of a broader network automation initiative. Manual console-based VPC creation introduces inconsistency, is not auditable via version control, and cannot be reliably repeated across environments.

**Requirements defined for this task:**

- Create an AWS VPC named `xfusion-vpc` with CIDR block `10.0.0.0/16`
- Store the VPC name in a Terraform input variable named `KKE_vpc`
- Separate configuration values into a `variables.tf` file
- Reference those variables from a `main.tf` resource definition
- Execute within the working directory `/home/bob/terraform`

---

## Solution Architecture

```
/home/bob/terraform/
├── provider.tf       # AWS provider configuration with LocalStack endpoints
├── variables.tf      # Input variable declarations (KKE_vpc)
├── main.tf           # Resource definition referencing variables
└── README.MD         # Original task specification
```

**Data flow:**

```
variables.tf (KKE_vpc = "xfusion-vpc")
        |
        v
main.tf (aws_vpc.main) --> provider.tf (LocalStack endpoint) --> VPC Created
```

The `provider.tf` file was pre-configured to redirect all AWS API calls to a LocalStack instance at `http://aws:4566`, with credential validation and account ID requests intentionally skipped for sandbox compatibility.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Terraform | Installed and accessible in `$PATH` |
| AWS Provider | `hashicorp/aws` version `5.91.0` |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` with `provider.tf` pre-existing |
| Terminal Access | VS Code Integrated Terminal or equivalent shell |

---

## Repository Structure

| File | Purpose |
|---|---|
| `provider.tf` | Declares the AWS provider, pins the version, configures LocalStack endpoints for all services |
| `variables.tf` | Declares the `KKE_vpc` input variable with type, description, and default value |
| `main.tf` | Defines the `aws_vpc` resource referencing `var.KKE_vpc` for the Name tag |
| `.terraform.lock.hcl` | Auto-generated provider lock file; committed to version control for deterministic installs |

---

## Implementation Guide

### Step 1: Inspect the Working Directory

Before creating any new files, the existing working directory was audited to understand what was pre-configured.

```bash
ls -la
```

**Output confirmed:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 May 10 19:47 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

The directory contained `provider.tf` (pre-configured with LocalStack endpoints) and a `README.MD`. No `main.tf` or `variables.tf` existed yet.

The contents of `provider.tf` were then reviewed to confirm provider version, region, and all endpoint overrides:

```bash
cat provider.tf
```

**`provider.tf` contents (pre-existing, not modified):**

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

> **Screenshot:**

<img width="537" height="397" alt="image" src="https://github.com/user-attachments/assets/8a0b3387-27ee-49c5-a6a9-6db7c5cbc48b" />

---

### Step 2: Create the Variables File

A `variables.tf` file was created using a heredoc to declare the `KKE_vpc` input variable with its type, description, and default value.

```bash
cat > variables.tf << 'EOF'
variable "KKE_vpc" {
  description = "The name of the VPC"
  type        = string
  default     = "xfusion-vpc"
}
EOF
```

The file content was verified immediately after creation:

```bash
cat variables.tf
```

**Expected output:**

```hcl
variable "KKE_vpc" {
  description = "The name of the VPC"
  type        = string
  default     = "xfusion-vpc"
}
```

> **Screenshot:** `step-02-variables-tf-created-and-verified.png`

**Design rationale:** Declaring the VPC name as an input variable with a `default` value allows the value to be overridden at runtime using `-var` flags or `.tfvars` files without modifying source code, which is critical for multi-environment deployments.

---

### Step 3: Create the Main Configuration File

A `main.tf` file was created to define the `aws_vpc` resource. The VPC name tag references the `KKE_vpc` variable declared in `variables.tf`.

```bash
cat > main.tf << 'EOF'
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = var.KKE_vpc
  }
}
EOF
```

The file content was verified immediately after creation:

```bash
cat main.tf
```

**Expected output:**

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = var.KKE_vpc
  }
}
```

> **Screenshot:** `step-03-main-tf-created-and-verified.png`

**Design rationale:** The resource logical name `main` follows Terraform convention for singleton resources of a given type per module. The CIDR block `10.0.0.0/16` provides 65,536 IP addresses, which is the standard starting allocation for a primary VPC.

---

### Step 4: Verify the Working Directory

After creating both files, the directory was listed again to confirm all required files were present with correct timestamps.

```bash
ls -la
```

**Expected output:**

```
total 28
drwxr-xr-x 1 bob bob 4096 May 10 19:58 .
drwxr-x--- 1 bob bob 4096 May 10 19:47 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob   98 May 10 19:58 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-r--r-- 1 bob bob  114 May 10 19:56 variables.tf
```

All three Terraform configuration files (`provider.tf`, `variables.tf`, `main.tf`) were confirmed present before proceeding to initialization.

> **Screenshot:** `step-04-directory-structure-confirmed.png`

---

### Step 5: Initialize Terraform

The Terraform working directory was initialized to download the required provider plugin and generate the lock file.

```bash
terraform init
```

**Expected output (abridged):**

```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl ...

Terraform has been successfully initialized!
```

> **Screenshot:** `step-05-terraform-init-success.png`

**What this step does:**

- Downloads `hashicorp/aws` version `5.91.0` from the Terraform Registry
- Creates `.terraform/` directory containing the provider binary
- Generates `.terraform.lock.hcl` to pin exact provider versions for reproducibility

---

### Step 6: Validate the Configuration

The configuration was validated to catch any syntax errors or invalid references before planning.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

> **Screenshot:** `step-06-terraform-validate-success.png`

Validation was run twice as observed in the execution log, confirming idempotency of the validate step.

---

### Step 7: Plan the Infrastructure Changes

A dry-run was executed to preview all changes Terraform would make without applying them. This is a mandatory gate before `apply` in production workflows.

```bash
terraform plan
```

**Expected output (key section):**

```
Terraform will perform the following actions:

  # aws_vpc.main will be created
  + resource "aws_vpc" "main" {
      + cidr_block   = "10.0.0.0/16"
      + enable_dns_support = true
      + instance_tenancy   = "default"
      + tags = {
          + "Name" = "xfusion-vpc"
        }
      + tags_all = {
          + "Name" = "xfusion-vpc"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> **Screenshot:** `step-07-terraform-plan-output.png`

**Key observations from the plan:**

- Exactly one resource will be created (`aws_vpc.main`)
- The `Name` tag correctly resolved to `"xfusion-vpc"` from `var.KKE_vpc`
- Several attributes show `(known after apply)`, which is expected for values AWS assigns at creation time (VPC ID, ARN, route table IDs, etc.)
- Zero destructive actions (`0 to change, 0 to destroy`)

---

### Step 8: Apply the Infrastructure Changes

With the plan confirmed, the configuration was applied. Terraform prompted for explicit confirmation before executing.

```bash
terraform apply
```

At the confirmation prompt:

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Expected output after confirmation:**

```
aws_vpc.main: Creating...
aws_vpc.main: Creation complete after 1s [id=vpc-d6e2aed23dcdf81ae]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Screenshot:** `step-08-terraform-apply-complete.png`

The VPC was successfully provisioned with ID `vpc-d6e2aed23dcdf81ae` within approximately 1 second via the LocalStack endpoint.

---

## Configuration Reference

### `variables.tf`

```hcl
variable "KKE_vpc" {
  description = "The name of the VPC"
  type        = string
  default     = "xfusion-vpc"
}
```

| Attribute | Value |
|---|---|
| Variable Name | `KKE_vpc` |
| Type | `string` |
| Default | `xfusion-vpc` |
| Override Method | `-var="KKE_vpc=<value>"` or `.tfvars` file |

### `main.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = var.KKE_vpc
  }
}
```

| Attribute | Value |
|---|---|
| Resource Type | `aws_vpc` |
| Logical Name | `main` |
| CIDR Block | `10.0.0.0/16` |
| Name Tag Source | `var.KKE_vpc` |

### `provider.tf` (pre-configured)

| Attribute | Value |
|---|---|
| Provider | `hashicorp/aws` |
| Version | `5.91.0` |
| Region | `us-east-1` |
| Endpoint Override | `http://aws:4566` (LocalStack) |
| Credential Validation | Skipped (sandbox mode) |

---

## Best Practices Applied

**Variable abstraction over hardcoded values**
The VPC name is declared in `variables.tf` rather than hardcoded in `main.tf`. This enables runtime override without source modification, supporting multi-environment pipelines (dev, staging, production) from a single codebase.

**Heredoc file creation for precision**
Files were created using `cat > file << 'EOF' ... EOF` to ensure exact content is written without shell interpolation, reducing human error from manual editor input in terminal environments.

**Immediate post-write verification**
Every file created was immediately verified with `cat` before proceeding to the next step. This closes the gap between intent and actual file content, catching encoding issues, truncation, or shell expansion anomalies early.

**Plan before apply**
`terraform plan` was executed before `terraform apply` to review the complete diff of infrastructure changes. In production, plan output should be saved with `-out=plan.tfplan` and the saved plan passed to `apply` to guarantee that what was reviewed is exactly what is applied.

**Explicit confirmation at apply**
The `apply` step requires typing `yes` explicitly. This gate prevents accidental infrastructure changes and is preserved as-is rather than bypassed with `-auto-approve`, which should be reserved for fully automated, audited CI/CD pipelines only.

**Provider version pinning**
The `required_providers` block pins `hashicorp/aws` to exactly `5.91.0`. Combined with the `.terraform.lock.hcl` file committed to version control, this ensures every engineer and pipeline run uses an identical provider binary, eliminating environment drift.

**Separation of concerns across files**
Configuration is distributed across `provider.tf`, `variables.tf`, and `main.tf` rather than consolidated in a single file. This mirrors Terraform community convention, improves readability, and allows targeted editing without risk of inadvertently modifying unrelated configuration blocks.

---

## Lessons Learned

**LocalStack endpoint configuration must be complete before initialization**
All service endpoints in `provider.tf` must be declared before running `terraform init`. Attempting to add endpoints after initialization requires re-running `init`, and in some cases the provider cache must be cleared. Always confirm `provider.tf` is correct and complete as the first step.

**Variable naming conventions affect discoverability**
The variable `KKE_vpc` uses a task-specific prefix (`KKE_`) that does not follow conventional Terraform naming patterns (`vpc_name` would be more idiomatic). In production modules, variable names should reflect their purpose and be consistent with module interface conventions to aid documentation generation and IDE tooling.

**`(known after apply)` attributes are expected, not errors**
During `terraform plan`, a large number of VPC attributes show `(known after apply)`. New practitioners sometimes interpret this as incomplete configuration. These values (VPC ID, ARN, default security group, route table IDs) are assigned by AWS at resource creation time and cannot be known in advance. This is correct and expected behavior.

**`terraform validate` is not a substitute for `terraform plan`**
Validation confirms syntactic correctness and variable reference integrity. It does not contact the provider API and will not catch resource-level misconfigurations (invalid CIDR formats, duplicate resource names, quota violations). Always run `plan` before `apply`.

**Lock files belong in version control**
The `.terraform.lock.hcl` file generated during `init` must be committed to the repository. It records cryptographic hashes of the exact provider versions installed. Without it, a future `terraform init` on a different machine may install a different patch version of the provider, leading to subtle behavioral differences.

---

## Errors Encountered and Resolutions

No errors were encountered during this implementation. All commands executed cleanly in sequence:

| Command | Result |
|---|---|
| `terraform init` | Success, provider installed |
| `terraform validate` | Success, configuration valid |
| `terraform plan` | Success, 1 resource planned |
| `terraform apply` | Success, 1 resource created |

**Preventive measures that avoided errors:**

- Pre-verifying `provider.tf` content before creating dependent files ensured no provider misconfiguration would surface during `init`
- Using heredoc syntax (`<< 'EOF'`) with single-quoted delimiter prevented unintended shell variable expansion inside `variables.tf` and `main.tf`
- Running `cat` verification after each file creation caught any potential write issues before they propagated to subsequent steps
- Confirming directory contents with `ls -la` after all files were created ensured no file was missing or misnamed before `init` was invoked

---

*All steps reflect exact commands executed in sequence with zero omissions or modifications.*






<img width="524" height="337" alt="image" src="https://github.com/user-attachments/assets/3418e06d-87ae-427e-b008-07af1649743c" />

<img width="524" height="308" alt="image" src="https://github.com/user-attachments/assets/d8f75ee6-abda-4fe1-a2e6-97d25496a9a2" />
<img width="522" height="152" alt="image" src="https://github.com/user-attachments/assets/1123565b-58d5-4626-a21c-41665da34701" />
<img width="524" height="240" alt="image" src="https://github.com/user-attachments/assets/9ddf01c2-49c3-4603-96aa-ae40bc4efa5c" />
<img width="525" height="322" alt="image" src="https://github.com/user-attachments/assets/345eb35f-2950-4296-9bc1-20ec95a00a6c" />
<img width="523" height="329" alt="image" src="https://github.com/user-attachments/assets/6330c7fc-d413-4bcd-87f2-d3a191aef226" />
<img width="524" height="284" alt="image" src="https://github.com/user-attachments/assets/e96974e4-0804-426e-9565-58bb10731b64" />
<img width="524" height="319" alt="image" src="https://github.com/user-attachments/assets/d4d5b26c-395a-46d6-b588-693a48da32bf" />
<img width="525" height="275" alt="image" src="https://github.com/user-attachments/assets/8f3d626c-0704-4c74-8778-058446b0e2aa" />
<img width="541" height="329" alt="image" src="https://github.com/user-attachments/assets/38f8d926-e9ab-447f-b51e-1c79a85db70d" />
<img width="540" height="383" alt="image" src="https://github.com/user-attachments/assets/6d4ef6c3-f5b6-4cab-b798-c02d67d59c41" />
