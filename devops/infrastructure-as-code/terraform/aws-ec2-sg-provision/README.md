# Terraform AWS Security Group Provisioning

> Provisioning a named AWS Security Group using Terraform with input variable abstraction, targeting a LocalStack endpoint, within a structured IaC working directory.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify the Working Directory](#step-1-verify-the-working-directory)
  - [Step 2: Create the Variables Definition File](#step-2-create-the-variables-definition-file)
  - [Step 3: Create the Main Configuration File (Initial Attempt)](#step-3-create-the-main-configuration-file-initial-attempt)
  - [Step 4: Investigate and Resolve the Duplicate Provider Error](#step-4-investigate-and-resolve-the-duplicate-provider-error)
  - [Step 5: Rewrite the Main Configuration Without the Provider Block](#step-5-rewrite-the-main-configuration-without-the-provider-block)
  - [Step 6: Initialize the Terraform Working Directory](#step-6-initialize-the-terraform-working-directory)
  - [Step 7: Validate the Configuration](#step-7-validate-the-configuration)
  - [Step 8: Generate the Execution Plan](#step-8-generate-the-execution-plan)
  - [Step 9: Apply the Configuration](#step-9-apply-the-configuration)
- [File Reference](#file-reference)
- [Error Encountered and Resolution](#error-encountered-and-resolution)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Project Overview

The Nautilus DevOps team required an AWS Security Group to be provisioned through Terraform with a specific name, stored cleanly in an input variable. The Security Group name `datacenter-sg` is assigned through a Terraform variable named `KKE_sg`, promoting reusability and separation of configuration from resource logic.

This implementation follows a structured, file-separated Terraform pattern where the provider block, variable definitions, and resource declarations are maintained across dedicated files. The deployment targets a LocalStack-backed AWS simulation, with provider configurations already established in a pre-existing `provider.tf` file.

---

## Architecture and Design Intent

```
terraform/
├── provider.tf       # Pre-existing: AWS provider + LocalStack endpoint configuration
├── variables.tf      # Input variable definition for the Security Group name
├── main.tf           # Resource declaration referencing the input variable
└── .terraform/       # Generated: Provider plugin cache after init
```

**Variable abstraction** separates the Security Group name from the resource block, enabling the same configuration to be reused across environments by overriding the variable at runtime without touching resource logic.

**Single provider configuration** is enforced by keeping all provider-level settings in `provider.tf` only. Defining a provider in both `main.tf` and `provider.tf` causes a fatal duplicate provider error that blocks initialization.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | v1.x or later |
| AWS Provider | hashicorp/aws v5.91.0 (pinned in provider.tf) |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |

---

## Repository Structure

```
terraform/
├── provider.tf       # AWS provider configuration targeting LocalStack
├── variables.tf      # KKE_sg variable definition (default: "datacenter-sg")
├── main.tf           # aws_security_group resource referencing var.KKE_sg
└── .terraform.lock.hcl  # Provider lock file (generated on init)
```

---

## Implementation Guide

### Step 1: Verify the Working Directory

Before writing any configuration, confirm the active working directory matches the designated Terraform project path.

```bash
pwd
```

**Expected output:**

```
/home/bob/terraform
```

> Screenshot: Terminal showing `/home/bob/terraform` as the current directory

<img width="521" height="262" alt="image" src="https://github.com/user-attachments/assets/975c2a90-31d3-4a7c-a7e4-4faa2b2be972" />

---

### Step 2: Create the Variables Definition File

Create `variables.tf` to define the `KKE_sg` input variable. This variable holds the Security Group name and provides a default value of `datacenter-sg`.

```bash
cat > variables.tf << 'EOF'
variable "KKE_sg" {
  description = "Name of the AWS Security Group"
  type        = string
  default     = "datacenter-sg"
}
EOF
```

Verify the file contents:

```bash
cat variables.tf
```

**Expected output:**

```hcl
variable "KKE_sg" {
  description = "Name of the AWS Security Group"
  type        = string
  default     = "datacenter-sg"
}
```

> Screenshot: Terminal showing `cat variables.tf` with the variable block output

<img width="523" height="287" alt="image" src="https://github.com/user-attachments/assets/40f8cd34-8d18-4bee-a178-86b03750c5b8" />

---

### Step 3: Create the Main Configuration File (Initial Attempt)

The first version of `main.tf` was written with an inline `provider "aws"` block alongside the resource declaration.

```bash
cat > main.tf << 'EOF'
provider "aws" {
  region = "us-east-1"
}

resource "aws_security_group" "datacenter_sg" {
  name        = var.KKE_sg
  description = "Security Group provisioned by Terraform"

  tags = {
    Name = var.KKE_sg
  }
}
EOF
```

Verify the file contents:

```bash
cat main.tf
```

> Screenshot: Terminal showing `cat main.tf` with the provider and resource blocks

<img width="522" height="338" alt="image" src="https://github.com/user-attachments/assets/f0d26cb1-ded3-4323-b3d6-934dc6a0285d" />

---

### Step 4: Investigate and Resolve the Duplicate Provider Error

Running `terraform init` with the above `main.tf` produced an initialization failure due to a duplicate provider definition.

```bash
terraform init
```

**Error output:**

```
Error: Duplicate provider configuration

  on provider.tf line 10:
  10: provider "aws" {

A default (non-aliased) provider configuration for "aws" was already given at main.tf:1,1-15.
If multiple configurations are required, set the "alias" argument for alternative configurations.
```

**Root cause:**

A `provider.tf` file already existed in the working directory, containing a fully configured `provider "aws"` block with LocalStack endpoint overrides. Defining a second default (non-aliased) `provider "aws"` block inside `main.tf` created a conflict that Terraform cannot resolve at initialization time.

**Resolution:**

Inspect the existing `provider.tf` to understand the full provider configuration already in place:

```bash
cat provider.tf
```

**Contents of the pre-existing `provider.tf`:**

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

The correct resolution is to remove the `provider "aws"` block from `main.tf` entirely and keep only the resource declaration, deferring all provider configuration to `provider.tf`.

> Screenshot: Terminal showing the full `terraform init` error output and the contents of `provider.tf`

<img width="521" height="367" alt="image" src="https://github.com/user-attachments/assets/2da41054-aa1c-492f-80da-c327e793506c" />
<img width="536" height="365" alt="image" src="https://github.com/user-attachments/assets/3ae427d6-1781-4cbf-b57d-e434ab2db8f7" />

---

### Step 5: Rewrite the Main Configuration Without the Provider Block

Overwrite `main.tf` to contain only the `aws_security_group` resource block, with no provider definition.

```bash
cat > main.tf << 'EOF'
resource "aws_security_group" "datacenter_sg" {
  name        = var.KKE_sg
  description = "Security Group provisioned by Terraform"

  tags = {
    Name = var.KKE_sg
  }
}
EOF
```

Verify the updated file:

```bash
cat main.tf
```

**Expected output:**

```hcl
resource "aws_security_group" "datacenter_sg" {
  name        = var.KKE_sg
  description = "Security Group provisioned by Terraform"

  tags = {
    Name = var.KKE_sg
  }
}
```

> Screenshot: Terminal showing the corrected `main.tf` with only the resource block

---

### Step 6: Initialize the Terraform Working Directory

With the duplicate provider conflict resolved, run `terraform init` to download the pinned provider plugin and prepare the working directory.

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
selections it made above.

Terraform has been successfully initialized!
```

> Screenshot: Terminal showing successful `terraform init` output with provider installation confirmation

---

### Step 7: Validate the Configuration

Run `terraform validate` to confirm that the configuration syntax and internal references are correct before generating a plan.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

> Screenshot: Terminal showing `Success! The configuration is valid.`

---

### Step 8: Generate the Execution Plan

Run `terraform plan` to preview the infrastructure changes Terraform will apply. This confirms the Security Group will be created with the correct name from the variable.

```bash
terraform plan
```

**Expected output (key section):**

```
Terraform will perform the following actions:

  # aws_security_group.datacenter_sg will be created
  + resource "aws_security_group" "datacenter_sg" {
      + description            = "Security Group provisioned by Terraform"
      + name                   = "datacenter-sg"
      + tags                   = {
          + "Name" = "datacenter-sg"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> Screenshot: Terminal showing the full `terraform plan` output with the `+ create` annotation on the Security Group resource

---

### Step 9: Apply the Configuration

Apply the configuration using the `--auto-approve` flag to provision the Security Group without an interactive confirmation prompt.

```bash
terraform apply --auto-approve
```

**Expected output:**

```
aws_security_group.datacenter_sg: Creating...
aws_security_group.datacenter_sg: Creation complete after 1s [id=sg-d019d41921cf152e2]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> Screenshot: Terminal showing `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` with the assigned Security Group ID

---

## File Reference

### `variables.tf`

```hcl
variable "KKE_sg" {
  description = "Name of the AWS Security Group"
  type        = string
  default     = "datacenter-sg"
}
```

### `main.tf`

```hcl
resource "aws_security_group" "datacenter_sg" {
  name        = var.KKE_sg
  description = "Security Group provisioned by Terraform"

  tags = {
    Name = var.KKE_sg
  }
}
```

### `provider.tf` (pre-existing, not modified)

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

---

## Error Encountered and Resolution

### Duplicate Provider Configuration

**Error:**

```
Error: Duplicate provider configuration

  on provider.tf line 10:
  10: provider "aws" {

A default (non-aliased) provider configuration for "aws" was already given at main.tf:1,1-15.
```

**Root Cause:**

Terraform does not permit two default (non-aliased) provider blocks for the same provider within the same module. The working directory already contained a `provider.tf` file with a complete `provider "aws"` configuration targeting LocalStack. Writing an additional `provider "aws"` block in `main.tf` created an irreconcilable conflict that aborted initialization before any plugins could be downloaded.

**Resolution:**

Remove the `provider "aws"` block from `main.tf` entirely. Terraform automatically discovers all `.tf` files in the working directory and merges their configurations. Provider declarations belong in a dedicated `provider.tf` file. Resource files such as `main.tf` should contain only resource blocks and references to input variables.

---

## Best Practices Applied

* **Separation of concerns:** Provider configuration, variable definitions, and resource declarations are maintained in separate files (`provider.tf`, `variables.tf`, `main.tf`). This aligns with the Terraform community standard for module and root module layout.

* **Input variable abstraction:** The Security Group name is not hardcoded in the resource block. Storing it in a variable with a type constraint and description makes the configuration self-documenting and overridable at runtime via `-var` flags or `.tfvars` files.

* **Tagging discipline:** The `Name` tag on the Security Group mirrors the `name` argument, ensuring consistency between the resource identifier visible in AWS Console and the Terraform-managed tag.

* **Provider version pinning:** The `provider.tf` file pins the AWS provider to `5.91.0` via `required_providers`. This prevents unintended provider upgrades from introducing breaking changes in CI/CD or team environments.

* **Validate before plan:** Running `terraform validate` as a distinct step before `terraform plan` catches syntax and reference errors early, separating configuration correctness from infrastructure change assessment.

* **Lock file committed:** The `.terraform.lock.hcl` file generated after `terraform init` records the exact provider version selected. This file should be committed to version control to guarantee reproducible provider installations across all team members and pipeline runs.

---

## Lessons Learned

* **Always inspect existing files before writing new configuration.** The duplicate provider error was caused by assuming the working directory contained only the files explicitly created during the session. In a shared or pre-configured environment, running `ls` or `cat` on all existing `.tf` files before writing new ones prevents avoidable conflicts.

* **Terraform merges all `.tf` files in the directory.** There is no concept of a "primary" file. Every `.tf` file in the root module is treated as part of the same configuration surface. This means a `provider` block in `main.tf` and a `provider` block in `provider.tf` are both active simultaneously.

* **Initialization failures can mask root causes.** When `terraform init` fails, the error message references file and line numbers precisely. Reading the error output in full, rather than re-running the command, leads directly to the conflicting declaration.

* **Non-aliased vs aliased providers:** Terraform supports multiple configurations of the same provider through the `alias` argument. If a second AWS provider targeting a different region or endpoint is genuinely needed, it must declare `alias = "secondary"` and be referenced explicitly in resources via `provider = aws.secondary`. Without an alias, the second block is always a conflict.

---

## Outcome

The AWS Security Group `datacenter-sg` was successfully provisioned against the LocalStack endpoint using a clean, production-structured Terraform configuration. The resource was created with the correct name, description, and `Name` tag as required, with configuration values fully abstracted through an input variable.

| Attribute | Value |
|---|---|
| Resource Type | `aws_security_group` |
| Security Group Name | `datacenter-sg` |
| Variable Name | `KKE_sg` |
| Terraform Resource ID | `sg-d019d41921cf152e2` |
| Provider Version | hashicorp/aws v5.91.0 |
| Apply Result | 1 added, 0 changed, 0 destroyed |











<img width="521" height="266" alt="image" src="https://github.com/user-attachments/assets/4c8d0c6a-fbed-4f96-8439-c3d1e977d86d" />

<img width="523" height="353" alt="image" src="https://github.com/user-attachments/assets/f218479d-6ac3-456e-b9b5-1c4dfc8db157" />

<img width="522" height="335" alt="image" src="https://github.com/user-attachments/assets/d91a11dc-0a9d-48c7-ae75-facc24a33343" />
<img width="523" height="365" alt="image" src="https://github.com/user-attachments/assets/d2ae2429-cf78-4743-a8f2-8411bd0e0f9a" />
<img width="524" height="295" alt="image" src="https://github.com/user-attachments/assets/e9dd046f-04b5-42d6-9359-730902fb3cfa" />
<img width="527" height="329" alt="image" src="https://github.com/user-attachments/assets/edcedc56-4ab6-464a-9340-6ff50137f77c" />
<img width="525" height="368" alt="image" src="https://github.com/user-attachments/assets/0f14d954-1658-4ef7-ba69-7cbeb0d4e43a" />
<img width="525" height="368" alt="image" src="https://github.com/user-attachments/assets/aac75d44-776e-4ef0-ab74-bd2dbcc51ce1" />
