# Provisioning an AWS EC2 Instance with Terraform (IaC)

> **Project:** Nautilus DevOps Infrastructure Migration Initiative
> **Author:** Bob (IAC Server)
> **Environment:** LocalStack (AWS-compatible mock endpoint)
> **Working Directory:** `/home/bob/terraform`
> **Date Executed:** April 9, 2025

---

## Table of Contents

1. [Overview](#overview)
2. [Problem Statement](#problem-statement)
3. [Solution Architecture](#solution-architecture)
4. [Prerequisites](#prerequisites)
5. [Implementation Guide](#implementation-guide)
   - [Step 1: Create the RSA Key Pair](#step-1-create-the-rsa-key-pair)
   - [Step 2: Write the Initial main.tf (First Attempt)](#step-2-write-the-initial-maintf-first-attempt)
   - [Step 3: Run terraform init (First Attempt - Error Encountered)](#step-3-run-terraform-init-first-attempt---error-encountered)
   - [Step 4: Investigate the Conflict Root Cause](#step-4-investigate-the-conflict-root-cause)
   - [Step 5: Rewrite main.tf (Corrected Version)](#step-5-rewrite-maintf-corrected-version)
   - [Step 6: Initialize, Validate, and Apply](#step-6-initialize-validate-and-apply)
6. [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
7. [Best Practices Applied](#best-practices-applied)
8. [Lessons Learned](#lessons-learned)
9. [File Reference](#file-reference)

---

## Overview

This document provides a precise, production-grade walkthrough of provisioning an AWS EC2 instance using Terraform as part of the Nautilus DevOps team's phased infrastructure migration to AWS. The implementation uses an IaC-first approach to ensure reproducibility, auditability, and team handoff readiness.

All infrastructure is defined as code. The instance is launched against a LocalStack endpoint simulating AWS services, which is appropriate for lab and pre-production validation workflows.

---

## Problem Statement

The Nautilus DevOps team is migrating a portion of their infrastructure to AWS. To manage complexity, the migration has been broken into incremental, task-scoped deliverables. This task covers the following specific requirements:

- Launch an EC2 instance tagged `datacenter-ec2`
- Use the Amazon Linux AMI `ami-0c101f26f147fa7fd`
- Set the instance type to `t2.micro`
- Associate a new RSA key pair named `datacenter-kp`
- Attach the default VPC security group
- All infrastructure must be defined exclusively in `main.tf` inside `/home/bob/terraform`

---

## Solution Architecture

```
/home/bob/terraform/
├── provider.tf          # Pre-existing: AWS provider config pointing to LocalStack
├── main.tf              # Authored in this task: EC2 data source + resource definition
└── datacenter-kp.pem    # RSA private key generated via AWS CLI
```

**Provider:** The pre-existing `provider.tf` configures the `hashicorp/aws` provider at version `5.91.0`, with all service endpoints pointing to LocalStack at `http://aws:4566`. This file must not be modified.

**Resource:** A single `aws_instance` resource references a data source that dynamically resolves the default security group ID, keeping the configuration decoupled from hardcoded values.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | Installed and available in `$PATH` |
| AWS CLI | Configured and accessible on the IaC server |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` with `provider.tf` already present |

---

## Implementation Guide

### Step 1: Create the RSA Key Pair

Before writing any Terraform configuration, the RSA key pair is created using the AWS CLI. The private key material is saved directly to a `.pem` file, and permissions are immediately locked down to owner-read-only (`400`) as required for SSH key security.

```bash
aws ec2 create-key-pair \
  --key-name datacenter-kp \
  --key-type rsa \
  --query "KeyMaterial" \
  --output text > datacenter-kp.pem
```

Verify the file was created:

```bash
ls -lh datacenter-kp.pem
```

**Expected output:**
```
-rw-r--r-- 1 bob bob 1.7K Apr  9 01:39 datacenter-kp.pem
```

Immediately restrict permissions:

```bash
chmod 400 datacenter-kp.pem && ls -lh datacenter-kp.pem
```

**Expected output:**
```
-r-------- 1 bob bob 1.7K Apr  9 01:39 datacenter-kp.pem
```

> **Screenshot:** 

<img width="1222" height="479" alt="image" src="https://github.com/user-attachments/assets/9ceb453a-29df-4bbf-bab6-64afbcc5ad53" />

---

### Step 2: Write the Initial main.tf (First Attempt)

The initial `main.tf` was written with the full Terraform block, provider block, and resource definition included in a single file:

```bash
cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_security_group" "default" {
  filter {
    name   = "group-name"
    values = ["default"]
  }
}

resource "aws_instance" "datacenter" {
  ami                    = "ami-0c101f26f147fa7fd"
  instance_type          = "t2.micro"
  key_name               = "datacenter-kp"
  vpc_security_group_ids = [data.aws_security_group.default.id]

  tags = {
    Name = "datacenter-ec2"
  }
}
EOF
```

Confirm file contents:

```bash
cat main.tf
```

> **Screenshot:** `screenshot_02_initial_maintf_written.png`

---

### Step 3: Run terraform init (First Attempt - Error Encountered)

```bash
terraform init
```

**Result:** Initialization failed with multiple errors.

```
Error: Duplicate required providers configuration
  on provider.tf line 2, in terraform:
   2:   required_providers {
A module may have only one required providers configuration.
The required providers were previously configured at main.tf:2,3-21.

Error: Duplicate provider configuration
  on provider.tf line 10:
   10: provider "aws" {
A default (non-aliased) provider configuration for "aws" was already given
at main.tf:10,1-15.
```

> **Screenshot:** `screenshot_03_terraform_init_error.png`

---

### Step 4: Investigate the Conflict Root Cause

Inspection of the working directory revealed a pre-existing `provider.tf` file:

```bash
ls -lh /home/bob/terraform/
```

**Output:**
```
total 16K
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-r-------- 1 bob bob 1.7K Apr  9 01:39 datacenter-kp.pem
-rw-r--r-- 1 bob bob  548 Apr  9 01:43 main.tf
-rw-rw-r-- 1 bob bob 1.1K May 13  2025 provider.tf
```

Review the contents of `provider.tf`:

```bash
cat provider.tf
```

**Contents:**
```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.91.0"
    }
  }
}

provider "aws" {
  region                      = "us-east-1"
  skip_credentials_validation = true
  skip_requesting_account_id  = true
  s3_use_path_style = true

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

**Root Cause:** Terraform treats all `.tf` files in a directory as a single module. Having `terraform {}` and `provider "aws" {}` blocks in both `main.tf` and `provider.tf` creates duplicate block declarations, which Terraform does not permit. Additionally, the version constraint in `main.tf` (`~> 4.0`) conflicted with the pinned version in `provider.tf` (`5.91.0`).

**Resolution:** Remove the `terraform {}` and `provider "aws" {}` blocks from `main.tf` entirely, retaining only the data source and resource definitions. The authoritative provider configuration must remain solely in `provider.tf`.

> **Screenshot:** `screenshot_04_provider_tf_inspection.png`

---

### Step 5: Rewrite main.tf (Corrected Version)

`main.tf` is overwritten to contain only the data source and resource blocks, eliminating any duplication with `provider.tf`:

```bash
cat > main.tf << 'EOF'
data "aws_security_group" "default" {
  filter {
    name   = "group-name"
    values = ["default"]
  }
}

resource "aws_instance" "datacenter" {
  ami                    = "ami-0c101f26f147fa7fd"
  instance_type          = "t2.micro"
  key_name               = "datacenter-kp"
  vpc_security_group_ids = [data.aws_security_group.default.id]

  tags = {
    Name = "datacenter-ec2"
  }
}
EOF
```

Verify the corrected file:

```bash
cat main.tf
```

> **Screenshot:** `screenshot_05_corrected_maintf.png`

---

### Step 6: Initialize, Validate, and Apply

All three Terraform workflow commands are executed in sequence:

```bash
terraform init
terraform validate
terraform apply -auto-approve
```

**terraform init output (success):**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "5.91.0"...
- Installing hashicorp/aws v5.91.0...
- Installed hashicorp/aws v5.91.0 (signed by HashiCorp)
Terraform has created a lock file .terraform.lock.hcl...
Terraform has been successfully initialized!
```

**terraform validate output:**
```
Success! The configuration is valid.
```

**terraform apply output (abridged):**
```
data.aws_security_group.default: Reading...
data.aws_security_group.default: Read complete after 0s [id=sg-6aa94959fe3dc15e6]

Terraform will perform the following actions:

  # aws_instance.datacenter will be created
  + resource "aws_instance" "datacenter" {
      + ami           = "ami-0c101f26f147fa7fd"
      + instance_type = "t2.micro"
      + key_name      = "datacenter-kp"
      + tags          = {
          + "Name" = "datacenter-ec2"
        }
      + vpc_security_group_ids = [
          + "sg-6aa94959fe3dc15e6",
        ]
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.

aws_instance.datacenter: Creating...
aws_instance.datacenter: Still creating... [10s elapsed]
aws_instance.datacenter: Creation complete after 10s [id=i-08cbec183209045a5]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Provisioned Instance Summary:**

| Attribute | Value |
|---|---|
| Instance ID | `i-08cbec183209045a5` |
| AMI | `ami-0c101f26f147fa7fd` |
| Instance Type | `t2.micro` |
| Key Pair | `datacenter-kp` |
| Security Group ID | `sg-6aa94959fe3dc15e6` |
| Name Tag | `datacenter-ec2` |

> **Screenshot:** `screenshot_06_terraform_apply_complete.png`

---

## Errors Encountered and Resolutions

### Error: Duplicate required_providers and Duplicate provider Configuration

| Field | Detail |
|---|---|
| **File** | `provider.tf` lines 2 and 10 |
| **Error Type** | `Duplicate required providers configuration` and `Duplicate provider configuration` |
| **Root Cause** | The initial `main.tf` included both a `terraform { required_providers {} }` block and a `provider "aws" {}` block. A pre-existing `provider.tf` in the same directory contained identical top-level blocks. Terraform loads all `.tf` files in a directory as one unified module and does not allow more than one default configuration per block type. |
| **Resolution** | Rewrote `main.tf` to contain only the `data` source and `resource` blocks. All provider and backend configuration was left exclusively in `provider.tf`. |
| **Prevention** | Before writing any new `.tf` file in a shared or pre-configured Terraform directory, always inspect existing files with `ls -lh` and `cat` to identify any pre-existing provider or backend blocks. |

---

## Best Practices Applied

* **Key pair permissions hardened immediately:** After generating `datacenter-kp.pem`, `chmod 400` was applied in the same command chain to ensure the private key was never left world-readable, even briefly.

* **Dynamic security group resolution via data source:** Rather than hardcoding a security group ID, a `data "aws_security_group"` block queries the actual group at plan time. This makes the configuration portable across accounts and environments where the default security group ID differs.

* **Provider configuration kept separate from resource definitions:** Provider and backend configuration lives exclusively in `provider.tf`. Resource and data source definitions live exclusively in `main.tf`. This separation of concerns aligns with production Terraform project conventions and avoids the duplicate block error encountered in this task.

* **Configuration validated before apply:** `terraform validate` was run as an explicit gate between `init` and `apply`, catching any syntactic or semantic issues before changes were submitted to the provider.

* **Version pinned in lock file:** Using an exact version (`5.91.0`) in `provider.tf` rather than a range constraint ensures deterministic provider behavior across all team members and CI runs. The generated `.terraform.lock.hcl` enforces this at the team level.

* **`-auto-approve` used intentionally in a lab context:** The apply was run with `-auto-approve` as this is a controlled lab environment. In production workflows, interactive approval or a separate plan artifact review step is required.

---

## Lessons Learned

* **Always audit existing `.tf` files before adding new ones.** Terraform silently merges all `.tf` files in a module directory. Any duplicate top-level block (e.g., `terraform {}`, `provider "aws" {}`) will cause `terraform init` to fail before any provider is even downloaded. The fix is trivial once identified, but the root cause is non-obvious without directory inspection.

* **The error message is precise and actionable.** Terraform's duplicate provider error clearly identifies the conflicting file and line number. Reading it carefully points directly to `provider.tf:2` and `provider.tf:10` as the sources of conflict, confirming that a pre-existing file was the issue rather than a syntax error in `main.tf`.

* **`terraform init` validates configuration before downloading providers.** Initialization does not proceed past the configuration validation phase if structural errors exist. This is a feature, not a bug, it prevents partial initialization states and ensures the working directory is clean.

* **Version constraints matter.** The initial `main.tf` used `~> 4.0` while `provider.tf` pinned to `5.91.0`. Even if the duplicate block error had not occurred, these constraints would have conflicted. When joining an existing Terraform project, always match the provider version already in use.

---

## File Reference

### provider.tf (Pre-existing, not modified)

Defines the AWS provider targeting LocalStack endpoints. Must not be duplicated in any other `.tf` file in the module.

### main.tf (Authored in this task)

```hcl
data "aws_security_group" "default" {
  filter {
    name   = "group-name"
    values = ["default"]
  }
}

resource "aws_instance" "datacenter" {
  ami                    = "ami-0c101f26f147fa7fd"
  instance_type          = "t2.micro"
  key_name               = "datacenter-kp"
  vpc_security_group_ids = [data.aws_security_group.default.id]

  tags = {
    Name = "datacenter-ec2"
  }
}
```

### datacenter-kp.pem (Generated via AWS CLI)

RSA private key for SSH access to the provisioned instance. Permissions set to `400`. Never commit this file to version control. Add it to `.gitignore`.

---

*Documentation produced following enterprise IaC documentation standards. Verified against the exact execution log recorded on April 9, 2025.*


<img width="1229" height="548" alt="image" src="https://github.com/user-attachments/assets/e3a522a7-fb10-4a3f-8e2d-ffbb73a8b8c4" />
<img width="1222" height="522" alt="image" src="https://github.com/user-attachments/assets/c1ab8a4b-2e3b-4061-8a71-ca79f3ebf036" />

<img width="1224" height="733" alt="image" src="https://github.com/user-attachments/assets/20bded03-8681-4b0e-bfe0-240a81ee703c" />
<img width="1223" height="732" alt="image" src="https://github.com/user-attachments/assets/2ecca87d-04be-4971-aa95-749d8a942c56" />
<img width="1260" height="739" alt="image" src="https://github.com/user-attachments/assets/cf6d29f0-428b-40ac-b161-e14b209e5f31" />
<img width="1254" height="254" alt="image" src="https://github.com/user-attachments/assets/c2f51b7b-67b4-48b3-bf2e-e3e0590ff854" />
<img width="1253" height="697" alt="image" src="https://github.com/user-attachments/assets/54d4c6fa-e455-4b74-84de-f99077c02f1e" />
<img width="1223" height="734" alt="image" src="https://github.com/user-attachments/assets/d25fa508-3dbc-4b86-a822-1d26e5933435" />
<img width="1225" height="739" alt="image" src="https://github.com/user-attachments/assets/972d2667-b92d-4272-a07f-b75148966d1c" />
<img width="1221" height="736" alt="image" src="https://github.com/user-attachments/assets/aef04df1-409f-4734-88fd-7db8d1b72dd7" />
<img width="1218" height="744" alt="image" src="https://github.com/user-attachments/assets/7ee0bfcd-b6d6-4ec8-8e44-9ad5d7422885" />
<img width="1206" height="740" alt="image" src="https://github.com/user-attachments/assets/9c2ffd6a-72cc-4e0b-8c11-b130638ea490" />
<img width="1224" height="740" alt="image" src="https://github.com/user-attachments/assets/f85571e0-9869-4d33-8217-792451062716" />

