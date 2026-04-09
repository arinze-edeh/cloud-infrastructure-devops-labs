# Terraform AWS Elastic IP Provisioning via LocalStack

> **Platform:** AWS (LocalStack-emulated) | **IaC Tool:** Terraform v1.11.0 | **Provider:** hashicorp/aws v5.91.0 | **Environment:** Nautilus DevOps / Stratos Datacenter

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Terraform Version and Working Directory](#step-1-verify-terraform-version-and-working-directory)
  - [Step 2: Create the Initial main.tf](#step-2-create-the-initial-maintf)
  - [Step 3: Verify the Written Configuration](#step-3-verify-the-written-configuration)
  - [Step 4: Inspect provider.tf and Identify the Conflict](#step-4-inspect-providertf-and-identify-the-conflict)
  - [Step 5: Rewrite main.tf to Resolve the Conflict](#step-5-rewrite-maintf-to-resolve-the-conflict)
  - [Step 6: Initialize the Terraform Working Directory](#step-6-initialize-the-terraform-working-directory)
  - [Step 7: Validate the Configuration](#step-7-validate-the-configuration)
  - [Step 8: Generate and Review the Execution Plan](#step-8-generate-and-review-the-execution-plan)
  - [Step 9: Apply the Configuration](#step-9-apply-the-configuration)
  - [Step 10: Verify State and Resource Attributes](#step-10-verify-state-and-resource-attributes)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Overview

This project documents the end-to-end Terraform workflow used to provision an **AWS Elastic IP (EIP)** address named `devops-eip` within a LocalStack-emulated AWS environment. The implementation is part of a structured, incremental infrastructure migration strategy executed by the Nautilus DevOps team. The working directory is `/home/bob/terraform`, and the resource definition is isolated in `main.tf`, intentionally separate from the pre-existing `provider.tf` configuration.

---

## Problem Statement

The Nautilus DevOps team is executing a phased migration of infrastructure components to AWS. Rather than performing a single large-scale cutover, the team has adopted a granular, task-by-task approach to minimize risk and disruption to ongoing operations.

This specific task requires allocating an **Elastic IP address** named `devops-eip` using **Terraform** within the designated working directory `/home/bob/terraform`. The Elastic IP serves as a static, publicly routable address that can be associated with EC2 instances, NAT Gateways, or network interfaces during subsequent migration phases.

The task mandates that:
- Only a `main.tf` file is created (no other `.tf` files)
- The pre-existing `provider.tf` must not be modified
- All provisioning is performed against the LocalStack-emulated AWS endpoint

---

## Architecture and Design Intent

```
/home/bob/terraform/
├── provider.tf        # Pre-existing: AWS provider targeting LocalStack (port 4566)
├── main.tf            # Authored: aws_eip resource definition
├── README.MD          # Pre-existing: Task context
└── .terraform/        # Generated: Provider plugin cache after terraform init
    └── .terraform.lock.hcl
```

**LocalStack** serves as the local AWS cloud emulator, exposing all AWS service endpoints on `http://aws:4566`. This allows full Terraform workflows to execute without live AWS credentials or incurring cloud costs, making it ideal for lab-based validation and team onboarding.

The `aws_eip` resource allocates a VPC-domain Elastic IP. In AWS provider v5.x, the `domain` attribute is resolved automatically by the provider as `"vpc"` without requiring explicit declaration, superseding the deprecated `vpc = true` boolean used in earlier provider versions.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Terraform | v1.11.0 (installed at `/usr/local/bin/terraform`) |
| AWS Provider | hashicorp/aws v5.91.0 (defined in `provider.tf`) |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |
| Shell Access | VS Code Integrated Terminal or SSH session as `bob` |

---

## Repository Structure

```
/home/bob/terraform/
├── provider.tf          # AWS provider config with LocalStack endpoint overrides
├── main.tf              # Elastic IP resource definition (authored in this task)
└── README.MD            # Original task brief
```

---

## Implementation Guide

### Step 1: Verify Terraform Version and Working Directory

Before authoring any configuration, confirm the installed Terraform version and validate the working directory context.

```bash
terraform version
```

**Output:**
```
Terraform v1.11.0
on linux_amd64

Your version of Terraform is out of date! The latest version
is 1.14.8. You can update by downloading from https://www.terraform.io/downloads.html
```

```bash
pwd
```

**Output:**
```
/home/bob/terraform
```

```bash
ls -la
```

**Output:**
```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr  8 23:13 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> **Screenshot:** 

<img width="1059" height="511" alt="image" src="https://github.com/user-attachments/assets/874aab56-fb4b-4cdd-84f5-abb9cad0dca5" />

The directory contains `provider.tf` and `README.MD`. No `main.tf` exists yet. The task requires creating `main.tf` as the sole new file.

---

### Step 2: Create the Initial main.tf

With the working directory confirmed, create `main.tf` using a heredoc. The initial attempt authored a self-contained configuration including a `terraform` block, a `provider` block, and the `aws_eip` resource.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
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

resource "aws_eip" "devops_eip" {
  vpc = true

  tags = {
    Name = "devops-eip"
  }
}
EOF
```

> **Screenshot:** 

<img width="1053" height="749" alt="image" src="https://github.com/user-attachments/assets/a87ccb75-e497-4bbb-9b81-6d782994208f" />

---

### Step 3: Verify the Written Configuration

Immediately after writing the file, verify its contents to confirm the heredoc was captured correctly.

```bash
cat /home/bob/terraform/main.tf
```

**Output:**
```hcl
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

resource "aws_eip" "devops_eip" {
  vpc = true

  tags = {
    Name = "devops-eip"
  }
}
```

> **Screenshot:** 

<img width="1049" height="656" alt="image" src="https://github.com/user-attachments/assets/4e2918d4-6399-4b75-883f-48d53fde3e5e" />

The file was written correctly. At this point the configuration had not yet been cross-checked against the pre-existing `provider.tf`.

---

### Step 4: Inspect provider.tf and Identify the Conflict

With `main.tf` written, inspect the pre-existing `provider.tf` to understand the provider version, region, and LocalStack endpoint mappings already declared in the root module.

```bash
cat /home/bob/terraform/provider.tf
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

> **Screenshot:** 

<img width="1050" height="762" alt="image" src="https://github.com/user-attachments/assets/eafd13bf-8964-4434-ae5c-b1f3d7a0efb9" />

Inspecting `provider.tf` revealed three problems with the `main.tf` written in Step 2:

* **Duplicate `terraform` block:** `provider.tf` already declares `required_providers`. Terraform merges all `.tf` files in the directory into a single root module, so a second `terraform` block in `main.tf` creates a collision.
* **Incompatible version constraint:** `main.tf` used `version = "~> 4.0"`, which would reject the v5.91.0 provider already pinned in `provider.tf`.
* **Duplicate `provider "aws"` block:** `provider.tf` already configures the AWS provider with LocalStack endpoints. A second bare `provider "aws"` block in `main.tf` is redundant and conflicting.
* **Deprecated `vpc = true` attribute:** AWS provider v5.x deprecates the `vpc` boolean on `aws_eip`. The provider now resolves `domain = "vpc"` automatically.

`main.tf` must be rewritten to contain only the resource block.

---

### Step 5: Rewrite main.tf to Resolve the Conflict

Overwrite `main.tf` with a corrected, minimal configuration containing only the `aws_eip` resource block. All provider and backend configuration remains exclusively in `provider.tf`.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_eip" "devops_eip" {
  tags = {
    Name = "devops-eip"
  }
}
EOF
```

Verify the rewritten content:

```bash
cat /home/bob/terraform/main.tf
```

**Output:**
```hcl
resource "aws_eip" "devops_eip" {
  tags = {
    Name = "devops-eip"
  }
}
```

> **Screenshot:** 

<img width="1047" height="744" alt="image" src="https://github.com/user-attachments/assets/ce685fda-2152-43f6-aa8c-aaa0e0971743" />

`main.tf` now contains only the resource definition. The `terraform` block, `provider` block, `vpc = true`, and the conflicting version constraint have all been removed. Together, `main.tf` and `provider.tf` form the complete root module.

---

### Step 6: Initialize the Terraform Working Directory

Run `terraform init` to download the required provider plugin (hashicorp/aws v5.91.0) and prepare the working directory.

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
```

> **Screenshot:**


<img width="1048" height="673" alt="image" src="https://github.com/user-attachments/assets/8e14bbe0-6cfc-4b8a-bbf6-0c59d2f60c3c" />

The provider was downloaded and installed successfully. A `.terraform.lock.hcl` file was generated to pin the provider version for reproducible future initializations.

---

### Step 7: Validate the Configuration

Run `terraform validate` to confirm that the configuration is syntactically correct and internally consistent before planning.

```bash
terraform validate
```

**Output:**
```
Success! The configuration is valid.
```

> **Screenshot:** 

<img width="1044" height="756" alt="image" src="https://github.com/user-attachments/assets/1a40cd15-807b-4b6c-a013-97888e0dc1dc" />

Validation passed with no errors or warnings. The resource block syntax is correct and the provider reference resolves cleanly.

---

### Step 8: Generate and Review the Execution Plan

Run `terraform plan` to preview the changes Terraform will make. This step is critical for verifying intent before any infrastructure mutation occurs.

```bash
terraform plan
```

**Output (abridged):**
```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_eip.devops_eip will be created
  + resource "aws_eip" "devops_eip" {
      + allocation_id        = (known after apply)
      + arn                  = (known after apply)
      + domain               = (known after apply)
      + public_ip            = (known after apply)
      + tags                 = {
          + "Name" = "devops-eip"
        }
      + tags_all             = {
          + "Name" = "devops-eip"
        }
      + vpc                  = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> **Screenshot:** 

<img width="1071" height="683" alt="image" src="https://github.com/user-attachments/assets/25aa057e-22ab-4978-bcd6-267d810aafb6" />

The plan confirms exactly one resource will be created: `aws_eip.devops_eip`. All computed attributes (allocation ID, ARN, public IP, domain) are marked as `(known after apply)`, which is expected for EIP resources where AWS assigns values at creation time.

---

### Step 9: Apply the Configuration

Apply the plan using the `-auto-approve` flag to suppress the interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Output (abridged):**
```
...
Plan: 1 to add, 0 to change, 0 to destroy.
aws_eip.devops_eip: Creating...
aws_eip.devops_eip: Creation complete after 1s [id=eipalloc-83a0519cfa9727816]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Screenshot:**

<img width="1075" height="683" alt="image" src="https://github.com/user-attachments/assets/e9fe0086-d361-4033-9e94-f26c3eab2857" />

The EIP was successfully allocated. LocalStack assigned the allocation ID `eipalloc-83a0519cfa9727816` and the apply completed in approximately one second.

---

### Step 10: Verify State and Resource Attributes

Confirm the resource is tracked in Terraform state and inspect the full attribute set.

**List managed resources:**

```bash
terraform state list
```

**Output:**
```
aws_eip.devops_eip
```

**Inspect full resource attributes:**

```bash
terraform state show aws_eip.devops_eip
```

**Output:**
```hcl
# aws_eip.devops_eip:
resource "aws_eip" "devops_eip" {
    allocation_id            = "eipalloc-83a0519cfa9727816"
    arn                      = "arn:aws:ec2:us-east-1::elastic-ip/eipalloc-83a0519cfa9727816"
    association_id           = null
    carrier_ip               = null
    customer_owned_ip        = null
    customer_owned_ipv4_pool = null
    domain                   = "vpc"
    id                       = "eipalloc-83a0519cfa9727816"
    instance                 = null
    network_border_group     = null
    network_interface        = null
    private_ip               = null
    ptr_record               = null
    public_dns               = "ec2-127-46-83-146.compute-1.amazonaws.com"
    public_ip                = "127.46.83.146"
    public_ipv4_pool         = null
    tags                     = {
        "Name" = "devops-eip"
    }
    tags_all                 = {
        "Name" = "devops-eip"
    }
    vpc                      = true
}
```

> **Screenshot:** 

<img width="1050" height="687" alt="image" src="https://github.com/user-attachments/assets/1834ddfb-ec5c-440b-a3ef-b483faa65e8f" />

**Verification checklist:**

| Attribute | Expected | Actual |
|---|---|---|
| Resource name tag | `devops-eip` | `devops-eip` |
| Domain | `vpc` | `vpc` |
| Allocation ID | Assigned by LocalStack | `eipalloc-83a0519cfa9727816` |
| Public IP | Assigned by LocalStack | `127.46.83.146` |
| Public DNS | Assigned by LocalStack | `ec2-127-46-83-146.compute-1.amazonaws.com` |
| State tracked | Yes | `aws_eip.devops_eip` confirmed |

All attributes reflect a healthy, fully allocated VPC-domain Elastic IP.

---

## Errors and Resolutions

### Error 1: Duplicate Provider and Terraform Block in main.tf

**Description:**
The initial version of `main.tf` included a full `terraform { required_providers { ... } }` block with `version = "~> 4.0"` and a standalone `provider "aws"` block. Since `provider.tf` already declares these at the root module level, having both files define the same provider caused an implicit conflict. Additionally, the `~> 4.0` version constraint would have rejected the v5.91.0 provider defined in `provider.tf`.

**Root Cause:**
The initial `main.tf` was authored as a self-contained configuration, not accounting for the pre-existing `provider.tf` in the same directory. Terraform merges all `.tf` files in a directory into a single root module, so duplicate top-level blocks collide.

**Resolution:**
Removed the `terraform` block and `provider "aws"` block entirely from `main.tf`, leaving only the `resource "aws_eip"` declaration. Provider configuration is exclusively managed by `provider.tf`.

---

### Error 2: Deprecated `vpc = true` Attribute

**Description:**
The initial `main.tf` used `vpc = true` inside the `aws_eip` resource. In AWS provider v5.x, this attribute is deprecated. The provider now automatically defaults all EIPs to the `vpc` domain and surfaces `domain = "vpc"` in state without requiring explicit declaration.

**Root Cause:**
The attribute syntax was carried over from provider v4.x patterns. The v5.x provider changelog deprecates `vpc = true` in favour of the implicit `domain` resolution.

**Resolution:**
Removed the `vpc = true` line from the resource block. The provider sets `domain = "vpc"` automatically and the state output confirms `vpc = true` for backward compatibility.

---

## Best Practices Applied

* **Separation of concerns:** Provider configuration (`provider.tf`) and resource definitions (`main.tf`) are maintained in separate files. This mirrors real-world project structure where provider settings, backend config, and resources each occupy dedicated files.

* **Exact provider version pinning:** The `provider.tf` uses `version = "5.91.0"` (exact pin) rather than a range constraint. This ensures deterministic behaviour across all team members and CI/CD pipelines without unexpected upgrades.

* **Validate before plan, plan before apply:** The `terraform validate` > `terraform plan` > `terraform apply` sequence was followed strictly. Running validate surfaces syntax errors without provider API calls; running plan surfaces logical errors without state mutation.

* **State verification post-apply:** `terraform state list` and `terraform state show` were executed after apply to confirm the resource is tracked correctly and all expected attributes are populated. This closes the feedback loop and validates the operation end-to-end.

* **Lock file committed:** The `.terraform.lock.hcl` generated by `terraform init` pins the provider to the exact downloaded version and hash. This file must be committed to version control for reproducible initializations.

* **Minimal resource definition:** The `aws_eip` block contains only the `tags` argument. No optional arguments that Terraform can resolve automatically (such as `domain`) are hard-coded, reducing brittleness and provider-version sensitivity.

---

## Lessons Learned

* **A Terraform root module is the entire directory, not a single file.** All `.tf` files in the same directory are merged before evaluation. Authoring `main.tf` without first reviewing `provider.tf` led to block duplication. Always inspect all existing `.tf` files before introducing new ones.

* **AWS provider v4 to v5 introduces breaking attribute changes.** The `vpc = true` attribute on `aws_eip` is a concrete example of a silent deprecation that does not fail loudly but should be removed for forward compatibility. When working with a pinned provider version, reviewing the provider changelog is as important as reviewing the Terraform docs.

* **LocalStack endpoint routing requires the `ec2` endpoint override.** EIP allocation uses the EC2 API surface. The `provider.tf` already included `ec2 = "http://aws:4566"`, which is why the apply succeeded without additional configuration. Missing this endpoint would have caused the API call to target real AWS, failing credential checks.

* **`-auto-approve` is appropriate in lab and CI contexts only.** For production workflows, the interactive confirmation step in `terraform apply` is a critical safety gate. In labs, `-auto-approve` streamlines the workflow, but the habit of always running `terraform plan` and reviewing the diff before applying is essential to carry into production practice.

* **`terraform state show` is the most reliable post-apply verification method.** It reads directly from the Terraform state file rather than re-querying the API, making it fast and provider-agnostic. It confirms both that the resource exists in state and that all attribute values are populated correctly.

---

## Outcome

An AWS Elastic IP address named `devops-eip` was successfully provisioned using Terraform against a LocalStack-emulated AWS environment. The EIP was allocated to the VPC domain with allocation ID `eipalloc-83a0519cfa9727816` and public IP `127.46.83.146`. The resource is tracked in Terraform state under the address `aws_eip.devops_eip` and is ready for association with EC2 instances or network interfaces in subsequent migration phases.

| Item | Value |
|---|---|
| Resource address | `aws_eip.devops_eip` |
| Allocation ID | `eipalloc-83a0519cfa9727816` |
| Public IP | `127.46.83.146` |
| Domain | `vpc` |
| Name tag | `devops-eip` |
| Provider version | hashicorp/aws v5.91.0 |
| Terraform version | v1.11.0 |
| Target environment | LocalStack (`http://aws:4566`) |














# Terraform AWS Elastic IP Provisioning via LocalStack

> **Platform:** AWS (LocalStack-emulated) | **IaC Tool:** Terraform v1.11.0 
---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Design Intent](#design-intent)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Terraform Version and Working Directory](#step-1-verify-terraform-version-and-working-directory)
  - [Step 2: Inspect the Existing Provider Configuration](#step-2-inspect-the-existing-provider-configuration)
  - [Step 3: Author the main.tf Resource File](#step-3-author-the-maintf-resource-file)
  - [Step 4: Reconcile Provider Configuration Conflict](#step-4-reconcile-provider-configuration-conflict)
  - [Step 5: Initialize the Terraform Working Directory](#step-5-initialize-the-terraform-working-directory)
  - [Step 6: Validate the Configuration](#step-6-validate-the-configuration)
  - [Step 7: Generate and Review the Execution Plan](#step-7-generate-and-review-the-execution-plan)
  - [Step 8: Apply the Configuration](#step-8-apply-the-configuration)
  - [Step 9: Verify State and Resource Attributes](#step-9-verify-state-and-resource-attributes)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Overview

This project documents the end-to-end Terraform workflow used to provision an **AWS Elastic IP (EIP)** address named `devops-eip` within a LocalStack-emulated AWS environment. The implementation is part of a structured, incremental infrastructure migration strategy executed by the Nautilus DevOps team. The working directory is `/home/bob/terraform`, and the resource definition is isolated in `main.tf`, intentionally separate from the pre-existing `provider.tf` configuration.

---

## Problem Statement

The Nautilus DevOps team is executing a phased migration of infrastructure components to AWS. Rather than performing a single large-scale cutover, the team has adopted a granular, task-by-task approach to minimize risk and disruption to ongoing operations.

This specific task requires allocating an **Elastic IP address** named `devops-eip` using **Terraform** within the designated working directory `/home/bob/terraform`. The Elastic IP serves as a static, publicly routable address that can be associated with EC2 instances, NAT Gateways, or network interfaces during subsequent migration phases.

The task mandates that:
- Only a `main.tf` file is created (no other `.tf` files)
- The pre-existing `provider.tf` must not be modified
- All provisioning is performed against the LocalStack-emulated AWS endpoint

---

## Design Intent

**LocalStack** serves as the local AWS cloud emulator, exposing all AWS service endpoints on `http://aws:4566`. This allows full Terraform workflows to execute without live AWS credentials or incurring cloud costs, making it ideal for lab-based validation and team onboarding.

The `aws_eip` resource allocates a VPC-domain Elastic IP. In AWS provider v5.x, the `domain` attribute is resolved automatically by the provider as `"vpc"` without requiring explicit declaration, superseding the deprecated `vpc = true` boolean used in earlier provider versions.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Terraform | v1.11.0 (installed at `/usr/local/bin/terraform`) |
| AWS Provider | hashicorp/aws v5.91.0 (defined in `provider.tf`) |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |
| Shell Access | VS Code Integrated Terminal or SSH session as `bob` |

---

## Implementation Guide

### Step 1: Verify Terraform Version and Working Directory

Before authoring any configuration, confirm the installed Terraform version and validate the working directory context.

```bash
terraform version
```

**Output:**
```
Terraform v1.11.0
on linux_amd64

Your version of Terraform is out of date! The latest version
is 1.14.8. You can update by downloading from https://www.terraform.io/downloads.html
```

```bash
pwd
```

**Output:**
```
/home/bob/terraform
```

```bash
ls -la
```

**Output:**
```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr  8 23:13 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> **Screenshot:**



The directory contains `provider.tf` and `README.MD`. No `main.tf` exists yet. The task requires creating `main.tf` as the sole new file.

---

### Step 2: Inspect the Existing Provider Configuration

Before writing any resource configuration, inspect the pre-existing `provider.tf` to understand the provider version, region, and LocalStack endpoint mappings.

```bash
cat /home/bob/terraform/provider.tf
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

> **Screenshot:** 



Key observations from `provider.tf`:
- Provider version is pinned to **5.91.0** (exact version, not a range constraint)
- `skip_credentials_validation = true` and `skip_requesting_account_id = true` are set to bypass AWS credential checks, appropriate for LocalStack
- All AWS service endpoints are redirected to `http://aws:4566`
- The `ec2` endpoint is included, which is required for EIP allocation

---

### Step 3: Author the main.tf Resource File

Create the `main.tf` file containing only the `aws_eip` resource definition. The task explicitly requires using `main.tf` and no other `.tf` file.

**Initial attempt** (subsequently revised):

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
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

resource "aws_eip" "devops_eip" {
  vpc = true

  tags = {
    Name = "devops-eip"
  }
}
EOF
```

> **Screenshot:** `03-initial-maintf-creation.png`

This initial version includes a duplicate `terraform` block and `provider` block that conflict with `provider.tf`, and also uses `version = "~> 4.0"` which is incompatible with the v5.91.0 provider already defined in `provider.tf`. It was immediately identified as problematic and replaced in the next step.

---

### Step 4: Reconcile Provider Configuration Conflict

The initial `main.tf` duplicated the `terraform` and `provider` blocks already declared in `provider.tf`. Terraform treats a single working directory as one root module, meaning duplicate block declarations cause conflicts. Additionally:

- The `vpc = true` attribute is **deprecated** in AWS provider v5.x. The `domain` attribute is now inferred automatically.
- A version constraint of `~> 4.0` would conflict with the `5.91.0` pin in `provider.tf`.

The file was rewritten to contain **only the resource block**, keeping `main.tf` clean and non-conflicting:

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_eip" "devops_eip" {
  tags = {
    Name = "devops-eip"
  }
}
EOF
```

Verify the final content:

```bash
cat /home/bob/terraform/main.tf
```

**Output:**
```hcl
resource "aws_eip" "devops_eip" {
  tags = {
    Name = "devops-eip"
  }
}
```

> **Screenshot:** 



This is the correct, minimal resource definition. The `provider.tf` file retains all provider configuration. The two files together form the complete root module.

---

### Step 5: Initialize the Terraform Working Directory

Run `terraform init` to download the required provider plugin (hashicorp/aws v5.91.0) and prepare the working directory.

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
```

> **Screenshot:** 



The provider was downloaded and installed successfully. A `.terraform.lock.hcl` file was generated to pin the provider version for reproducible future initializations.

---

### Step 6: Validate the Configuration

Run `terraform validate` to confirm that the configuration is syntactically correct and internally consistent before planning.

```bash
terraform validate
```

**Output:**
```
Success! The configuration is valid.
```

> **Screenshot:** 



Validation passed with no errors or warnings. The resource block syntax is correct and the provider reference resolves cleanly.

---

### Step 7: Generate and Review the Execution Plan

Run `terraform plan` to preview the changes Terraform will make. This step is critical for verifying intent before any infrastructure mutation occurs.

```bash
terraform plan
```

**Output (abridged):**
```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_eip.devops_eip will be created
  + resource "aws_eip" "devops_eip" {
      + allocation_id        = (known after apply)
      + arn                  = (known after apply)
      + domain               = (known after apply)
      + public_ip            = (known after apply)
      + tags                 = {
          + "Name" = "devops-eip"
        }
      + tags_all             = {
          + "Name" = "devops-eip"
        }
      + vpc                  = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> **Screenshot:** 



The plan confirms exactly one resource will be created: `aws_eip.devops_eip`. All computed attributes (allocation ID, ARN, public IP, domain) are marked as `(known after apply)`, which is expected for EIP resources where AWS assigns values at creation time.

---

### Step 8: Apply the Configuration

Apply the plan using the `-auto-approve` flag to suppress the interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Output (abridged):**
```
...
Plan: 1 to add, 0 to change, 0 to destroy.
aws_eip.devops_eip: Creating...
aws_eip.devops_eip: Creation complete after 1s [id=eipalloc-83a0519cfa9727816]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Screenshot:** 



The EIP was successfully allocated. LocalStack assigned the allocation ID `eipalloc-83a0519cfa9727816` and the apply completed in approximately one second.

---

### Step 9: Verify State and Resource Attributes

Confirm the resource is tracked in Terraform state and inspect the full attribute set.

**List managed resources:**

```bash
terraform state list
```

**Output:**
```
aws_eip.devops_eip
```

**Inspect full resource attributes:**

```bash
terraform state show aws_eip.devops_eip
```

**Output:**
```hcl
# aws_eip.devops_eip:
resource "aws_eip" "devops_eip" {
    allocation_id            = "eipalloc-83a0519cfa9727816"
    arn                      = "arn:aws:ec2:us-east-1::elastic-ip/eipalloc-83a0519cfa9727816"
    association_id           = null
    carrier_ip               = null
    customer_owned_ip        = null
    customer_owned_ipv4_pool = null
    domain                   = "vpc"
    id                       = "eipalloc-83a0519cfa9727816"
    instance                 = null
    network_border_group     = null
    network_interface        = null
    private_ip               = null
    ptr_record               = null
    public_dns               = "ec2-127-46-83-146.compute-1.amazonaws.com"
    public_ip                = "127.46.83.146"
    public_ipv4_pool         = null
    tags                     = {
        "Name" = "devops-eip"
    }
    tags_all                 = {
        "Name" = "devops-eip"
    }
    vpc                      = true
}
```

> **Screenshot:** 



**Verification checklist:**

| Attribute | Expected | Actual |
|---|---|---|
| Resource name tag | `devops-eip` | `devops-eip` |
| Domain | `vpc` | `vpc` |
| Allocation ID | Assigned by LocalStack | `eipalloc-83a0519cfa9727816` |
| Public IP | Assigned by LocalStack | `127.46.83.146` |
| Public DNS | Assigned by LocalStack | `ec2-127-46-83-146.compute-1.amazonaws.com` |
| State tracked | Yes | `aws_eip.devops_eip` confirmed |

All attributes reflect a healthy, fully allocated VPC-domain Elastic IP.

---

## Errors and Resolutions

### Error 1: Duplicate Provider and Terraform Block in main.tf

**Description:**
The initial version of `main.tf` included a full `terraform { required_providers { ... } }` block with `version = "~> 4.0"` and a standalone `provider "aws"` block. Since `provider.tf` already declares these at the root module level, having both files define the same provider caused an implicit conflict. Additionally, the `~> 4.0` version constraint would have rejected the v5.91.0 provider defined in `provider.tf`.

**Root Cause:**
The initial `main.tf` was authored as a self-contained configuration, not accounting for the pre-existing `provider.tf` in the same directory. Terraform merges all `.tf` files in a directory into a single root module, so duplicate top-level blocks collide.

**Resolution:**
Removed the `terraform` block and `provider "aws"` block entirely from `main.tf`, leaving only the `resource "aws_eip"` declaration. Provider configuration is exclusively managed by `provider.tf`.

---

### Error 2: Deprecated `vpc = true` Attribute

**Description:**
The initial `main.tf` used `vpc = true` inside the `aws_eip` resource. In AWS provider v5.x, this attribute is deprecated. The provider now automatically defaults all EIPs to the `vpc` domain and surfaces `domain = "vpc"` in state without requiring explicit declaration.

**Root Cause:**
The attribute syntax was carried over from provider v4.x patterns. The v5.x provider changelog deprecates `vpc = true` in favour of the implicit `domain` resolution.

**Resolution:**
Removed the `vpc = true` line from the resource block. The provider sets `domain = "vpc"` automatically and the state output confirms `vpc = true` for backward compatibility.

---

## Best Practices Applied

* **Separation of concerns:** Provider configuration (`provider.tf`) and resource definitions (`main.tf`) are maintained in separate files. This mirrors real-world project structure where provider settings, backend config, and resources each occupy dedicated files.

* **Exact provider version pinning:** The `provider.tf` uses `version = "5.91.0"` (exact pin) rather than a range constraint. This ensures deterministic behaviour across all team members and CI/CD pipelines without unexpected upgrades.

* **Validate before plan, plan before apply:** The `terraform validate` > `terraform plan` > `terraform apply` sequence was followed strictly. Running validate surfaces syntax errors without provider API calls; running plan surfaces logical errors without state mutation.

* **State verification post-apply:** `terraform state list` and `terraform state show` were executed after apply to confirm the resource is tracked correctly and all expected attributes are populated. This closes the feedback loop and validates the operation end-to-end.

* **Lock file committed:** The `.terraform.lock.hcl` generated by `terraform init` pins the provider to the exact downloaded version and hash. This file must be committed to version control for reproducible initializations.

* **Minimal resource definition:** The `aws_eip` block contains only the `tags` argument. No optional arguments that Terraform can resolve automatically (such as `domain`) are hard-coded, reducing brittleness and provider-version sensitivity.

---

## Lessons Learned

* **A Terraform root module is the entire directory, not a single file.** All `.tf` files in the same directory are merged before evaluation. Authoring `main.tf` without first reviewing `provider.tf` led to block duplication. Always inspect all existing `.tf` files before introducing new ones.

* **AWS provider v4 to v5 introduces breaking attribute changes.** The `vpc = true` attribute on `aws_eip` is a concrete example of a silent deprecation that does not fail loudly but should be removed for forward compatibility. When working with a pinned provider version, reviewing the provider changelog is as important as reviewing the Terraform docs.

* **LocalStack endpoint routing requires the `ec2` endpoint override.** EIP allocation uses the EC2 API surface. The `provider.tf` already included `ec2 = "http://aws:4566"`, which is why the apply succeeded without additional configuration. Missing this endpoint would have caused the API call to target real AWS, failing credential checks.

* **`-auto-approve` is appropriate in lab and CI contexts only.** For production workflows, the interactive confirmation step in `terraform apply` is a critical safety gate. In labs, `-auto-approve` streamlines the workflow, but the habit of always running `terraform plan` and reviewing the diff before applying is essential to carry into production practice.

* **`terraform state show` is the most reliable post-apply verification method.** It reads directly from the Terraform state file rather than re-querying the API, making it fast and provider-agnostic. It confirms both that the resource exists in state and that all attribute values are populated correctly.

---

## Outcome

An AWS Elastic IP address named `devops-eip` was successfully provisioned using Terraform against a LocalStack-emulated AWS environment. The EIP was allocated to the VPC domain with allocation ID `eipalloc-83a0519cfa9727816` and public IP `127.46.83.146`. The resource is tracked in Terraform state under the address `aws_eip.devops_eip` and is ready for association with EC2 instances or network interfaces in subsequent migration phases.

| Item | Value |
|---|---|
| Resource address | `aws_eip.devops_eip` |
| Allocation ID | `eipalloc-83a0519cfa9727816` |
| Public IP | `127.46.83.146` |
| Domain | `vpc` |
| Name tag | `devops-eip` |
| Provider version | hashicorp/aws v5.91.0 |
| Terraform version | v1.11.0 |
| Target environment | LocalStack (`http://aws:4566`) |















