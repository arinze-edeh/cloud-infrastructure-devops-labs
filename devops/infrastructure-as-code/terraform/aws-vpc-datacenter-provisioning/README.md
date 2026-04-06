# Terraform AWS VPC Provisioning

Provision a custom AWS Virtual Private Cloud (VPC) using Terraform IaC against a LocalStack-emulated AWS environment, as the foundational networking layer for an incremental cloud migration strategy.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: Inspect the Working Directory](#phase-1-inspect-the-working-directory)
  - [Phase 2: Review the Provider Configuration](#phase-2-review-the-provider-configuration)
  - [Phase 3: Author the VPC Resource Configuration](#phase-3-author-the-vpc-resource-configuration)
  - [Phase 4: Initialize the Terraform Working Directory](#phase-4-initialize-the-terraform-working-directory)
  - [Phase 5: Validate the Configuration](#phase-5-validate-the-configuration)
  - [Phase 6: Generate and Review the Execution Plan](#phase-6-generate-and-review-the-execution-plan)
  - [Phase 7: Apply the Configuration](#phase-7-apply-the-configuration)
  - [Phase 8: Verify State and Resource](#phase-8-verify-state-and-resource)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

The Nautilus DevOps team is migrating a portion of their infrastructure to AWS. Rather than executing a single large-scale cutover, the team adopted an incremental migration strategy, provisioning isolated Virtual Private Clouds (VPCs) as the initial networking foundation before onboarding individual services.

This document covers the Terraform-based provisioning of a VPC named `datacenter-vpc` in the `us-east-1` region with the IPv4 CIDR block `192.168.0.0/24`, executed entirely within the designated Terraform working directory `/home/bob/terraform`.

**Scope:**

* Provider: AWS (HashiCorp) v5.91.0
* Target resource: `aws_vpc`
* Environment: LocalStack (`http://aws:4566`) simulating real AWS endpoints
* CIDR: `192.168.0.0/24`
* VPC Name tag: `datacenter-vpc`

---

## Architecture

```
LocalStack (http://aws:4566)
        |
        | AWS API (emulated)
        |
Terraform (v5.91.0 AWS Provider)
        |
        | terraform apply
        |
aws_vpc.datacenter_vpc
  CIDR:  192.168.0.0/24
  Name:  datacenter-vpc
  ID:    vpc-25fc7ded2b37bd271
  Region: us-east-1
```

The provider is configured with LocalStack endpoint overrides for all relevant AWS services, enabling fully offline IaC development and testing without incurring cloud costs or requiring real AWS credentials.

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform CLI | Any version compatible with AWS provider 5.91.0 |
| AWS Provider | HashiCorp `hashicorp/aws` v5.91.0 |
| LocalStack | Running at `http://aws:4566` |
| Working directory | `/home/bob/terraform` |
| Existing file | `provider.tf` (pre-configured, do not modify) |

---

## Implementation

### Phase 1: Inspect the Working Directory

Confirmed the contents of the Terraform working directory before making any changes.

```bash
ls -la
```

**Output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr  6 03:08 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

The directory contains the pre-existing `provider.tf` and `README.MD` only. No `main.tf` exists yet.

*Screenshot: Terminal output of `ls -la` showing initial directory contents*

<img width="1150" height="516" alt="image" src="https://github.com/user-attachments/assets/ba32215a-7a6c-40d0-b305-2aef3c3d9245" />

---

### Phase 2: Review the Provider Configuration

Inspected `provider.tf` to understand the AWS provider configuration before authoring resources.

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
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` allow operation without real AWS credentials
* All service endpoints are redirected to LocalStack at `http://aws:4566`
* Region is locked to `us-east-1`

*Screenshot: Terminal output of `cat provider.tf` displaying the full provider block*

<img width="1135" height="766" alt="image" src="https://github.com/user-attachments/assets/91f93c24-1bd4-4fa1-99d6-ba81e21add6a" />

---

### Phase 3: Author the VPC Resource Configuration

Created `main.tf` using a heredoc to define the `aws_vpc` resource with the required CIDR and name tag. No additional `.tf` files were created.

```bash
cat > main.tf << 'EOF'
resource "aws_vpc" "datacenter_vpc" {
  cidr_block = "192.168.0.0/24"

  tags = {
    Name = "datacenter-vpc"
  }
}
EOF
```

Verified the file content:

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_vpc" "datacenter_vpc" {
  cidr_block = "192.168.0.0/24"

  tags = {
    Name = "datacenter-vpc"
  }
}
```

*Screenshot: Terminal output confirming `main.tf` contents after creation*

<img width="1115" height="767" alt="image" src="https://github.com/user-attachments/assets/4d9c9190-4f33-4545-9779-4721e9e3edd4" />

---

### Phase 4: Initialize the Terraform Working Directory

Ran `terraform init` to download the required provider plugin and initialize the backend.

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

The provider was installed and signed by HashiCorp. The `.terraform.lock.hcl` file was generated, pinning the provider version for reproducible runs.

*Screenshot: Terminal output of `terraform init` showing successful provider installation*

<img width="1160" height="754" alt="image" src="https://github.com/user-attachments/assets/4dd92fe6-1a4b-4dc0-aa39-31a2dd61b224" />

---

### Phase 5: Validate the Configuration

Validated the Terraform configuration for syntax and internal consistency before planning.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

*Screenshot: Terminal output of `terraform validate` confirming configuration validity*

<img width="1160" height="638" alt="image" src="https://github.com/user-attachments/assets/f2a12fa0-c642-4aa0-91f3-e03d52295c4b" />

---

### Phase 6: Generate and Review the Execution Plan

Generated a Terraform execution plan to preview all changes before applying them.

```bash
terraform plan
```

**Output (relevant excerpt):**

```
Terraform will perform the following actions:

  # aws_vpc.datacenter_vpc will be created
  + resource "aws_vpc" "datacenter_vpc" {
      + arn                                  = (known after apply)
      + cidr_block                           = "192.168.0.0/24"
      + enable_dns_support                   = true
      + instance_tenancy                     = "default"
      + tags                                 = {
          + "Name" = "datacenter-vpc"
        }
      + tags_all                             = {
          + "Name" = "datacenter-vpc"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

The plan confirmed that exactly one resource would be created with the intended CIDR block and name tag. No unintended changes were surfaced.

*Screenshot: Terminal output of `terraform plan` showing the full execution plan*

<img width="1155" height="761" alt="image" src="https://github.com/user-attachments/assets/88bf56f5-3857-4fed-bfb8-e3b0ca9fdc91" />

---

### Phase 7: Apply the Configuration

Applied the configuration with auto-approval to provision the VPC.

```bash
terraform apply -auto-approve
```

**Output (relevant excerpt):**

```
aws_vpc.datacenter_vpc: Creating...
aws_vpc.datacenter_vpc: Creation complete after 1s [id=vpc-25fc7ded2b37bd271]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The VPC was created in 1 second and assigned the ID `vpc-25fc7ded2b37bd271` by the LocalStack environment.

*Screenshot: Terminal output of `terraform apply -auto-approve` showing successful resource creation*

---

### Phase 8: Verify State and Resource

**Step 8a: List managed resources in Terraform state**

```bash
terraform state list
```

**Output:**

```
aws_vpc.datacenter_vpc
```

**Step 8b: Inspect full resource attributes from state**

```bash
terraform show
```

**Output:**

```hcl
# aws_vpc.datacenter_vpc:
resource "aws_vpc" "datacenter_vpc" {
    arn                                  = "arn:aws:ec2:us-east-1:000000000000:vpc/vpc-25fc7ded2b37bd271"
    assign_generated_ipv6_cidr_block     = false
    cidr_block                           = "192.168.0.0/24"
    default_network_acl_id               = "acl-df3bc153a896f869a"
    default_route_table_id               = "rtb-8ef4eb581b1c48bbe"
    default_security_group_id            = "sg-1dcd6ecb66cff7a99"
    dhcp_options_id                      = "default"
    enable_dns_hostnames                 = false
    enable_dns_support                   = true
    enable_network_address_usage_metrics = false
    id                                   = "vpc-25fc7ded2b37bd271"
    instance_tenancy                     = "default"
    main_route_table_id                  = "rtb-8ef4eb581b1c48bbe"
    owner_id                             = "000000000000"
    tags                                 = {
        "Name" = "datacenter-vpc"
    }
    tags_all                             = {
        "Name" = "datacenter-vpc"
    }
}
```

All expected attributes were confirmed:
* `cidr_block` = `192.168.0.0/24`
* `tags.Name` = `datacenter-vpc`
* `id` = `vpc-25fc7ded2b37bd271`
* Associated default network ACL, route table, and security group were automatically provisioned by AWS (LocalStack) upon VPC creation

*Screenshot: Terminal output of `terraform show` displaying full VPC state attributes*

---

## Key Decisions

**Single `main.tf` file for resource definitions**
The task explicitly required all resource declarations to reside in `main.tf` only. No additional `.tf` files (such as `variables.tf` or `outputs.tf`) were introduced, maintaining a clean, minimal file surface appropriate for this scope.

**Heredoc-based file creation (`cat > main.tf << 'EOF'`)**
Using a quoted heredoc (`<< 'EOF'`) guaranteed that no variable interpolation or shell expansion occurred inside the block. This is the safest method for writing multi-line HCL content directly from the terminal.

**`-auto-approve` on apply**
Applied with `-auto-approve` as the plan had been explicitly reviewed in the preceding step. In production workflows, the recommended pattern is `terraform plan -out=tfplan` followed by `terraform apply tfplan` to guarantee that only the reviewed plan is executed.

**Provider pre-configuration (no modification)**
The `provider.tf` was pre-configured with LocalStack endpoint overrides and was not modified. This separation of provider configuration from resource definitions follows the standard Terraform convention of isolating provider concerns from infrastructure resource declarations.

**CIDR selection: `192.168.0.0/24`**
The `/24` prefix provides 256 addresses (254 usable), sufficient for the initial migration phase. The `192.168.0.0` block is a standard RFC 1918 private range, appropriate for cloud VPC use and compatible with future VPC peering without CIDR conflicts, provided peered VPCs use non-overlapping ranges.

---

## Errors and Resolutions

No errors were encountered during this implementation. All phases completed successfully on the first attempt. The following potential failure modes are documented for team reference:

| Scenario | Likely Cause | Resolution |
|---|---|---|
| `Error: Failed to install provider` during `terraform init` | Network connectivity to the Terraform registry | Verify internet access or use a local provider mirror |
| `Error: Invalid provider configuration` | LocalStack not running at `http://aws:4566` | Confirm LocalStack service health before applying |
| `Error: CIDR block X.X.X.X/X is invalid` | Malformed CIDR string in `main.tf` | Validate CIDR notation with a subnet calculator before authoring |
| `terraform validate` fails with `Invalid block definition` | Syntax error in HCL | Review heredoc output with `cat main.tf` and correct bracket or key-value formatting |

---

## Lessons Learned

**Always separate plan from apply in production**
While `-auto-approve` is acceptable in controlled lab environments after an explicit plan review, production pipelines should always capture the plan output with `terraform plan -out=tfplan` and apply only the saved plan. This eliminates any drift between what was reviewed and what gets applied if infrastructure state changes between the two commands.

**`terraform show` is the authoritative post-apply verification tool**
`terraform state list` confirms that a resource exists in state, but `terraform show` is required to inspect actual attribute values. Using both together provides full post-apply confidence: existence check followed by attribute validation.

**LocalStack endpoint redirection enables cost-free IaC development**
Configuring the AWS provider to point all service endpoints at `http://aws:4566` allows the full Terraform workflow (init, validate, plan, apply, show) to execute identically to a real AWS environment without credentials or billing. This pattern is highly valuable for building and testing modules before promoting to production.

**AWS auto-provisions default VPC components on creation**
Upon VPC creation, AWS (and LocalStack faithfully emulating this behavior) automatically creates a default network ACL, a default route table, and a default security group. These appear in `terraform show` output under `default_network_acl_id`, `default_route_table_id`, and `default_security_group_id`. They are not Terraform-managed resources and will not appear in `terraform state list` unless explicitly imported.

**`dns_hostnames` is disabled by default**
The `terraform show` output confirmed `enable_dns_hostnames = false`. In VPCs that need to support EC2 instances accessible by public DNS names, this attribute must be explicitly set to `true` in the `aws_vpc` resource block. Omitting it is a common oversight that surfaces only when EC2 public DNS resolution is required downstream.










<img width="1148" height="776" alt="image" src="https://github.com/user-attachments/assets/5fc7114e-984d-44df-a131-a7011df4be87" />
<img width="1157" height="191" alt="image" src="https://github.com/user-attachments/assets/76347b7e-3900-4172-b859-ffe5bcab506b" />
<img width="1151" height="727" alt="image" src="https://github.com/user-attachments/assets/2d404a39-1613-48dc-80b4-dabe077d22bc" />
