# Terraform AWS VPC Provisioning with IPv6 Support

## Table of Contents

* [Overview](#overview)
* [Problem Statement](#problem-statement)
* [Solution Architecture](#solution-architecture)
* [Prerequisites](#prerequisites)
* [Implementation Guide](#implementation-guide)
  * [Phase 1: Environment Verification](#phase-1-environment-verification)
  * [Phase 2: Terraform Configuration Review](#phase-2-terraform-configuration-review)
  * [Phase 3: VPC Resource Definition](#phase-3-vpc-resource-definition)
  * [Phase 4: Provider Initialization](#phase-4-provider-initialization)
  * [Phase 5: Configuration Validation](#phase-5-configuration-validation)
  * [Phase 6: Infrastructure Provisioning](#phase-6-infrastructure-provisioning)
  * [Phase 7: State Verification](#phase-7-state-verification)
* [Key Decisions](#key-decisions)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)
* [Outcome](#outcome)

---

## Overview

This project provisions an AWS Virtual Private Cloud (VPC) using Terraform as the Infrastructure as Code (IaC) tool. The VPC is configured with a dual-stack network: an IPv4 CIDR block and an Amazon-assigned IPv6 CIDR block. The implementation targets a LocalStack-emulated AWS environment to support isolated, cost-free infrastructure development and testing.

This is the foundational networking layer for the Nautilus DevOps team's phased AWS cloud migration strategy, with VPCs serving as the isolation boundary for all subsequent service deployments.

---

## Problem Statement

The Nautilus DevOps team is executing a phased migration of existing infrastructure to AWS. The initial step requires establishing Virtual Private Clouds as the network boundary for each workload segment. The first VPC, `devops-vpc`, must be provisioned in the `us-east-1` region with both IPv4 and Amazon-assigned IPv6 addressing to support modern dual-stack workloads. All provisioning must be done through Terraform to enforce IaC discipline and reproducibility.

---

## Solution Architecture

| Attribute | Value |
|---|---|
| VPC Name | `devops-vpc` |
| IPv4 CIDR Block | `10.0.0.0/16` |
| IPv6 CIDR Block | Amazon-assigned (auto-generated) |
| Region | `us-east-1` |
| AWS Provider | `hashicorp/aws v5.91.0` |
| Terraform Version | `v1.11.0` |
| Execution Target | LocalStack (`http://aws:4566`) |
| IaC Entry Point | `main.tf` |

The provider routes all AWS API calls to a LocalStack endpoint, enabling full infrastructure simulation without incurring cloud costs or requiring live AWS credentials.

---

## Prerequisites

* Terraform `v1.11.0` or later installed on the execution host
* LocalStack running and accessible at `http://aws:4566`
* Write access to the `/home/bob/terraform` working directory
* Existing `provider.tf` with LocalStack endpoint configuration

---

## Implementation Guide

### Phase 1: Environment Verification

Confirm the working directory and Terraform binary version before any configuration changes.

```bash
pwd
```

Expected output confirms the working directory:

```
/home/bob/terraform
```

```bash
ls -la
```

Verify the pre-existing files are present:

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr  7 03:16 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

```bash
terraform version
```

```
Terraform v1.11.0
on linux_amd64
```

> Note: Terraform reports that v1.14.8 is available. This version is sufficient for the task and no upgrade is performed.

*Screenshot: Terminal showing working directory path, file listing, and Terraform version output*

<img width="1054" height="492" alt="image" src="https://github.com/user-attachments/assets/2dd8a83d-c2bb-4f07-8c8b-7ef038c835c5" />

---

### Phase 2: Terraform Configuration Review

Inspect the existing provider configuration to confirm the LocalStack endpoint mapping and AWS provider version before writing any resource definitions.

```bash
cat provider.tf
```

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
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack compatibility
* All service endpoints resolve to the LocalStack container at port `4566`
* The `ec2` endpoint is present, confirming VPC operations will be routed correctly

*Screenshot: Terminal output of provider.tf contents*

<img width="1194" height="766" alt="image" src="https://github.com/user-attachments/assets/e703a598-6187-422e-b453-7180777eaffd" />

---

### Phase 3: VPC Resource Definition

Create `main.tf` in the working directory using a heredoc to write the VPC resource block precisely and avoid interactive editor issues.

```bash
cat > main.tf << 'EOF'
resource "aws_vpc" "devops_vpc" {
  cidr_block                       = "10.0.0.0/16"
  assign_generated_ipv6_cidr_block = true

  tags = {
    Name = "devops-vpc"
  }
}
EOF
```

Verify the file was written correctly:

```bash
cat main.tf
```

```hcl
resource "aws_vpc" "devops_vpc" {
  cidr_block                       = "10.0.0.0/16"
  assign_generated_ipv6_cidr_block = true

  tags = {
    Name = "devops-vpc"
  }
}
```

Key attributes:
* `cidr_block`: Defines the IPv4 address space as `10.0.0.0/16`, providing 65,536 addresses
* `assign_generated_ipv6_cidr_block`: Instructs AWS to assign an Amazon-provided `/56` IPv6 CIDR block automatically
* `tags.Name`: Sets the human-readable identifier `devops-vpc` for console and CLI filtering

*Screenshot: Terminal showing the cat heredoc command and the resulting main.tf content*

<img width="1196" height="770" alt="image" src="https://github.com/user-attachments/assets/0895da3f-6fe2-401a-9dd1-bd6a2218c160" />

---

### Phase 4: Provider Initialization

Initialize the Terraform working directory to download the required provider plugin and generate the dependency lock file.

```bash
terraform init
```

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

The lock file `.terraform.lock.hcl` is generated and pins provider version `5.91.0` for reproducible deployments across all environments.

*Screenshot: Terminal showing terraform init completion with provider installation confirmation*

<img width="1195" height="586" alt="image" src="https://github.com/user-attachments/assets/50a9e79f-a61c-4361-945f-a6e033be5df8" />

---

### Phase 5: Configuration Validation

Run the Terraform validator to confirm the configuration is syntactically correct and internally consistent before applying.

```bash
terraform validate
```

```
Success! The configuration is valid.
```

*Screenshot: Terminal showing terraform validate success output*

<img width="1193" height="776" alt="image" src="https://github.com/user-attachments/assets/b8528f17-cb9b-4a5c-9dc6-9d15c655a2f4" />

---

### Phase 6: Infrastructure Provisioning

Apply the Terraform plan with auto-approval. Terraform computes the execution plan, displays all attributes to be created, and proceeds with resource creation.

```bash
terraform apply -auto-approve
```

Terraform generates and displays the full execution plan:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.devops_vpc will be created
  + resource "aws_vpc" "devops_vpc" {
      + arn                                  = (known after apply)
      + assign_generated_ipv6_cidr_block     = true
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = (known after apply)
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags                                 = {
          + "Name" = "devops-vpc"
        }
      + tags_all                             = {
          + "Name" = "devops-vpc"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_vpc.devops_vpc: Creating...
aws_vpc.devops_vpc: Still creating... [10s elapsed]
aws_vpc.devops_vpc: Creation complete after 11s [id=vpc-8088692a29dd38c1c]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The VPC was provisioned successfully with resource ID `vpc-8088692a29dd38c1c`. Creation completed in approximately 11 seconds, which reflects LocalStack's simulated provisioning latency.

*Screenshot: Terminal showing terraform apply output with plan, creation progress, and completion summary*

<img width="1195" height="766" alt="image" src="https://github.com/user-attachments/assets/761f1044-6da0-40a3-9f3a-4ae09bb97cc0" />

---

### Phase 7: State Verification

Inspect the Terraform state to confirm all resource attributes are populated correctly post-apply.

```bash
terraform show
```

```hcl
# aws_vpc.devops_vpc:
resource "aws_vpc" "devops_vpc" {
    arn                                  = "arn:aws:ec2:us-east-1:000000000000:vpc/vpc-8088692a29dd38c1c"
    assign_generated_ipv6_cidr_block     = true
    cidr_block                           = "10.0.0.0/16"
    default_network_acl_id               = "acl-d4642cd538abab9cf"
    default_route_table_id               = "rtb-627668a6d8fc8d7c2"
    default_security_group_id            = "sg-9ae534cc9f9e069e4"
    dhcp_options_id                      = "default"
    enable_dns_hostnames                 = false
    enable_dns_support                   = true
    enable_network_address_usage_metrics = false
    id                                   = "vpc-8088692a29dd38c1c"
    instance_tenancy                     = "default"
    ipv6_association_id                  = "vpc-cidr-assoc-ad25be320b8346e0b"
    ipv6_cidr_block                      = "2400:6500:6198:2400::/56"
    ipv6_netmask_length                  = 0
    main_route_table_id                  = "rtb-627668a6d8fc8d7c2"
    owner_id                             = "000000000000"
    tags                                 = {
        "Name" = "devops-vpc"
    }
    tags_all                             = {
        "Name" = "devops-vpc"
    }
}
```

All critical attributes are confirmed:

| Attribute | Expected | Actual |
|---|---|---|
| `cidr_block` | `10.0.0.0/16` | `10.0.0.0/16` |
| `assign_generated_ipv6_cidr_block` | `true` | `true` |
| `ipv6_cidr_block` | Amazon-assigned `/56` | `2400:6500:6198:2400::/56` |
| `ipv6_association_id` | Populated | `vpc-cidr-assoc-ad25be320b8346e0b` |
| `tags.Name` | `devops-vpc` | `devops-vpc` |
| `enable_dns_support` | `true` (default) | `true` |

*Screenshot: Terminal showing full terraform show output with all populated VPC attributes*

<img width="1191" height="693" alt="image" src="https://github.com/user-attachments/assets/4d47a383-edce-4f3e-a0ce-fed928613d1e" />

---

## Key Decisions

**Heredoc for file creation over interactive editors**
The `cat > main.tf << 'EOF'` heredoc pattern was used instead of `vi` or `nano` to write `main.tf`. In KodeKloud terminal environments, `vi` substitution behavior can be unreliable. The heredoc approach eliminates editor-specific quirks and makes the exact file content reproducible in documentation and automation.

**Single `main.tf` file, no additional `.tf` files**
The task specification explicitly required that only `main.tf` be created. Separating resources across multiple files (e.g., `vpc.tf`) was intentionally avoided to stay within the defined scope. In production environments with multiple resource types, separation by service domain is the preferred pattern.

**`assign_generated_ipv6_cidr_block = true` over manual IPv6 CIDR specification**
Using the Amazon-provided IPv6 CIDR block ensures compatibility with AWS's regional IPv6 allocation pools and avoids conflicts with custom IPAM configurations. This is the correct approach for standard VPC deployments that do not require Bring Your Own IP (BYOIP).

**`terraform apply -auto-approve` in a controlled environment**
Auto-approval is appropriate in isolated LocalStack environments where plan review overhead adds no safety value. In production pipelines, the plan step must always be a separate, reviewed gate before apply.

**`terraform show` for post-apply verification over `terraform state list`**
`terraform show` surfaces the full attribute set of all managed resources in a human-readable format, making it the most efficient single command for verifying the complete state of a provisioned resource.

---

## Errors and Resolutions

No errors were encountered during this implementation. The provider initialized cleanly, configuration validation passed on the first attempt, and resource provisioning completed successfully within the expected timeframe.

**Noted advisory (non-blocking):**
Terraform reported that v1.14.8 is available while v1.11.0 is installed. This is an informational notice only and does not affect functionality. In a production environment, Terraform version upgrades should be coordinated across all team members and CI pipelines to maintain state compatibility.

---

## Lessons Learned

**LocalStack provisioning latency is simulated, not instantaneous**
The VPC creation took approximately 11 seconds despite targeting LocalStack. Some LocalStack service simulations introduce artificial delays to more closely mirror real AWS behavior. Pipelines that target LocalStack for integration testing should account for this latency in timeout configurations.

**The `(known after apply)` pattern in Terraform plans reflects dynamic attribute resolution**
Attributes like `arn`, `default_route_table_id`, and `ipv6_cidr_block` cannot be computed until the resource exists in the provider. This is expected behavior. The `terraform show` output after apply captures the fully resolved state. Understanding this distinction is critical when writing dependent resource configurations that reference computed attributes using `resource.type.name.attribute` syntax.

**`terraform validate` catches structural errors, not provider-level semantic errors**
Validation confirmed the HCL syntax was correct, but it does not simulate the API call. Errors such as unsupported instance types, invalid CIDR ranges, or policy-restricted SKUs only surface during `terraform apply`. Always treat validation as a necessary but not sufficient pre-apply check.

**Separating provider configuration from resource definitions improves reusability**
Keeping LocalStack endpoint overrides in `provider.tf` and resource definitions in `main.tf` means the resource configuration is portable. Switching from LocalStack to a live AWS environment requires only a `provider.tf` change, with no modifications to resource definitions.

**The `tags.Name` attribute is the primary identifier for VPC discoverability**
AWS resource IDs are auto-generated and opaque. The `Name` tag is what surfaces in the AWS Console, CLI filter outputs (`--filters "Name=tag:Name,Values=devops-vpc"`), and cost allocation reports. Always set a `Name` tag on every VPC and its child resources from the initial provisioning step.

---

## Outcome

A fully functional AWS VPC named `devops-vpc` was provisioned via Terraform IaC into a LocalStack environment. The VPC is configured with a `10.0.0.0/16` IPv4 address space and an Amazon-assigned `/56` IPv6 CIDR block (`2400:6500:6198:2400::/56`), supporting dual-stack workloads. Default supporting constructs including a network ACL, route table, security group, and DHCP options set were automatically created by AWS as part of VPC initialization. The Terraform state file accurately reflects all resource attributes, and the configuration is ready for extension with subnets, internet gateways, and route table associations in subsequent provisioning phases.
