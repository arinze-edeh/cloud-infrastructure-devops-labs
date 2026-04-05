# Terraform AWS VPC Provisioning on LocalStack

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Working Directory and Existing Configuration](#step-1-verify-working-directory-and-existing-configuration)
  - [Step 2: Author the Main Terraform Configuration](#step-2-author-the-main-terraform-configuration)
  - [Step 3: Initialize the Terraform Working Directory](#step-3-initialize-the-terraform-working-directory)
  - [Step 4: Validate the Configuration](#step-4-validate-the-configuration)
  - [Step 5: Apply the Infrastructure Plan](#step-5-apply-the-infrastructure-plan)
  - [Step 6: Verify Provisioned State](#step-6-verify-provisioned-state)
- [Resource Specifications](#resource-specifications)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors and Resolutions](#errors-and-resolutions)

---

## Overview

This documents the first infrastructure unit of the Nautilus DevOps team's phased AWS cloud migration. Rather than executing a large-scale, monolithic migration, the team has adopted an incremental delivery strategy, decomposing the overall migration into discrete, independently verifiable infrastructure units. This stage provisions a foundational **Virtual Private Cloud (VPC)** using Terraform, targeting a LocalStack-emulated AWS environment to enable safe, cost-free validation before production deployment.

---

## Problem Statement

The Nautilus DevOps team is migrating a portion of their infrastructure to the AWS cloud. The scale of this undertaking introduced risk if approached as a single transition. To address this, the team segmented the migration into smaller, manageable units. This granular approach enables gradual execution, ensures smoother implementation, minimises disruption to ongoing operations, and allows for better control, risk mitigation, and resource optimisation throughout the migration process.

The immediate objective of this stage is to provision a VPC named **`devops-vpc`** in the **`us-east-1`** region with an IPv4 CIDR block, using Terraform, within a pre-existing working directory structure.

---

## Solution Architecture

```
LocalStack (http://aws:4566)
        |
        |-- EC2 / VPC API
              |
              +-- aws_vpc.devops_vpc
                    CIDR: 10.0.0.0/16
                    Tag:  Name = "devops-vpc"
                    Region: us-east-1
```

Terraform communicates exclusively with LocalStack endpoints, allowing all AWS API calls to be intercepted and emulated locally. This pattern is production-safe for development and staging validation.

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform | >= 1.x |
| AWS Provider | 5.91.0 (HashiCorp) |
| LocalStack | Running and accessible at `http://aws:4566` |
| AWS CLI (optional) | For manual endpoint verification |
| Working directory | `/home/bob/terraform` |

> **Note:** AWS credential validation is intentionally bypassed in the provider configuration (`skip_credentials_validation = true`, `skip_requesting_account_id = true`) because LocalStack does not enforce IAM authentication in its default configuration.

---

## Implementation Guide

### Step 1: Verify Working Directory and Existing Configuration

Before authoring any new configuration, confirm the state of the working directory and review the existing `provider.tf` to understand the LocalStack endpoint mappings.

```bash
ls -la
```

**Output observed:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr  5 02:50 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

Review the provider configuration:

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

> *Screenshots: Terminal output showing `ls -la` and `cat provider.tf` results confirming the pre-existing directory state.*

<img width="1037" height="403" alt="image" src="https://github.com/user-attachments/assets/d5ce5708-fc74-4631-a0a1-0267ba06ec9d" />
<img width="1252" height="764" alt="image" src="https://github.com/user-attachments/assets/983cb493-b9b5-457e-9fd1-628eefd29f2a" />

---

### Step 2: Author the Main Terraform Configuration

Create `main.tf` using a heredoc redirect. This is the only new file introduced in this task.

```bash
cat > main.tf << 'EOF'
resource "aws_vpc" "devops_vpc" {
  cidr_block = "10.0.0.0/16"

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

**Output observed:**

```hcl
resource "aws_vpc" "devops_vpc" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "devops-vpc"
  }
}
```

> *Screenshot: Terminal showing the heredoc command and subsequent `cat main.tf` output confirming correct file content.*

<img width="1150" height="774" alt="image" src="https://github.com/user-attachments/assets/ad5f7509-4e7e-4e65-bf7b-3109f6b555c6" />

**Configuration breakdown:**

| Field | Value | Purpose |
|---|---|---|
| Resource type | `aws_vpc` | Instructs Terraform to manage an AWS VPC resource |
| Resource name | `devops_vpc` | Terraform-internal logical identifier |
| `cidr_block` | `10.0.0.0/16` | IPv4 address space for the VPC (65,536 addresses) |
| `tags.Name` | `devops-vpc` | AWS console-visible name tag for the VPC |

---

### Step 3: Initialize the Terraform Working Directory

Run `terraform init` to download the required provider plugin and configure the backend.

```bash
terraform init
```

**Output observed:**

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

> *Screenshot: Terminal output from `terraform init` confirming successful provider installation and backend initialisation.*

<img width="1245" height="753" alt="image" src="https://github.com/user-attachments/assets/a3fa664b-85d3-4772-a385-7671a14044e0" />

This step produces a `.terraform/` directory and a `.terraform.lock.hcl` lock file. The lock file pins the provider to version `5.91.0`, ensuring deterministic behaviour across all future `terraform init` runs in this repository.

---

### Step 4: Validate the Configuration

Run `terraform validate` to perform a static syntax and schema check before any plan or apply is executed.

```bash
terraform validate
```

**Output observed:**

```
Success! The configuration is valid.
```

> *Screenshot: Terminal showing `terraform validate` returning a clean success status.*

<img width="1189" height="642" alt="image" src="https://github.com/user-attachments/assets/52856faf-5d7a-4f00-820f-d09422a19a7d" />

Validation confirms that the HCL syntax is well-formed and that all resource arguments conform to the AWS provider schema for `aws_vpc`.

---

### Step 5: Apply the Infrastructure Plan

Execute `terraform apply` with the `-auto-approve` flag to provision the VPC without requiring interactive confirmation. This is appropriate in a controlled LocalStack environment.

```bash
terraform apply -auto-approve
```

**Output observed:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_vpc.devops_vpc will be created
  + resource "aws_vpc" "devops_vpc" {
      + arn                                  = (known after apply)
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
aws_vpc.devops_vpc: Creation complete after 1s [id=vpc-a819a7443995df162]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> *Screenshot: Full terminal output of `terraform apply -auto-approve` showing the execution plan, creation progress, and final success summary.*

The VPC was successfully created and assigned the ID `vpc-a819a7443995df162` by LocalStack.

---

### Step 6: Verify Provisioned State

Confirm the resource was correctly recorded in Terraform state by inspecting the state entry for `aws_vpc.devops_vpc`.

```bash
terraform state show aws_vpc.devops_vpc
```

**Output observed:**

```hcl
# aws_vpc.devops_vpc:
resource "aws_vpc" "devops_vpc" {
    arn                                  = "arn:aws:ec2:us-east-1:000000000000:vpc/vpc-a819a7443995df162"
    assign_generated_ipv6_cidr_block     = false
    cidr_block                           = "10.0.0.0/16"
    default_network_acl_id               = "acl-66a69973524afa028"
    default_route_table_id               = "rtb-1ed23e9e7809a5231"
    default_security_group_id            = "sg-55e17d45593e3ed58"
    dhcp_options_id                      = "default"
    enable_dns_hostnames                 = false
    enable_dns_support                   = true
    enable_network_address_usage_metrics = false
    id                                   = "vpc-a819a7443995df162"
    instance_tenancy                     = "default"
    ipv6_association_id                  = null
    ipv6_cidr_block                      = null
    ipv6_cidr_block_network_border_group = null
    ipv6_ipam_pool_id                    = null
    ipv6_netmask_length                  = 0
    main_route_table_id                  = "rtb-1ed23e9e7809a5231"
    owner_id                             = "000000000000"
    tags                                 = {
        "Name" = "devops-vpc"
    }
    tags_all                             = {
        "Name" = "devops-vpc"
    }
}
```

> *Screenshot: Terminal output of `terraform state show aws_vpc.devops_vpc` confirming all resource attributes are correctly captured in state.*

**State attribute highlights:**

| Attribute | Value | Significance |
|---|---|---|
| `id` | `vpc-a819a7443995df162` | Unique VPC identifier assigned by LocalStack |
| `cidr_block` | `10.0.0.0/16` | Matches the declared configuration exactly |
| `tags.Name` | `devops-vpc` | Tag propagated correctly |
| `enable_dns_support` | `true` | AWS default; DNS resolution within the VPC is active |
| `enable_dns_hostnames` | `false` | AWS default; public DNS hostnames for instances are disabled until explicitly enabled |
| `instance_tenancy` | `default` | Shared hardware tenancy; cost-optimal baseline |
| `owner_id` | `000000000000` | LocalStack placeholder account ID |

---

## Resource Specifications

| Property | Value |
|---|---|
| Resource Type | `aws_vpc` |
| Terraform Logical Name | `devops_vpc` |
| VPC Name Tag | `devops-vpc` |
| CIDR Block | `10.0.0.0/16` |
| Region | `us-east-1` |
| Provider | HashiCorp AWS `5.91.0` |
| Endpoint | LocalStack (`http://aws:4566`) |
| VPC ID (provisioned) | `vpc-a819a7443995df162` |

---

## Best Practices Applied

***Single-responsibility file structure*** -- Resource definitions are authored exclusively in `main.tf`, keeping provider configuration isolated in `provider.tf`. This separation of concerns makes individual components easier to audit, update, and peer-review.

***Provider version pinning*** -- The AWS provider is locked to version `5.91.0` via both `required_providers` and the generated `.terraform.lock.hcl`. This eliminates the risk of upstream provider changes silently breaking infrastructure behaviour across team environments.

***Static validation before apply*** -- Running `terraform validate` prior to `terraform apply` creates a fast-feedback gate that catches syntax and schema errors without incurring any API calls or state mutations.

***Resource tagging*** -- The `Name` tag is applied directly to the VPC resource. In production AWS environments, consistent tagging is mandatory for cost allocation, resource discovery, and compliance auditing.

***LocalStack-first development*** -- All API calls are redirected to LocalStack, enabling complete infrastructure validation with zero AWS spend and zero risk of accidental production resource creation during development and testing phases.

***Heredoc for file authoring*** -- Using `cat > main.tf << 'EOF' ... EOF` ensures the file is written atomically in the terminal without relying on a text editor, making the authoring step fully reproducible and scriptable.

---

## Lessons Learned

**1. Understand `(known after apply)` values in plan output**
During `terraform plan` and `terraform apply`, many VPC attributes display as `(known after apply)`. These are server-side computed values such as the VPC ARN, default route table ID, and default security group ID. These are expected and do not indicate a problem. The full set of computed values is visible only after successful apply via `terraform state show`.

**2. `enable_dns_hostnames` defaults to `false`**
The state output revealed that `enable_dns_hostnames` is `false` by default. In subsequent stages where EC2 instances require public DNS resolution, this attribute must be explicitly set to `true` in the VPC resource block. Leaving it as a silent default in a production configuration is a common source of connectivity confusion.

**3. `terraform state show` is the authoritative post-apply verification step**
The apply output confirms creation but shows limited attribute detail. `terraform state show` exposes the full hydrated resource state, including all computed attributes. This should be a mandatory verification step after every `apply` in any environment.

**4. The `.terraform.lock.hcl` file must be committed to version control**
The lock file records the exact provider version and its cryptographic checksums. Committing it ensures that every team member and every CI pipeline uses the identical provider binary, eliminating version drift across environments.

**5. `-auto-approve` is appropriate only in controlled, non-production environments**
The `terraform apply -auto-approve` flag was used here because the target is a LocalStack emulator. In production pipelines, this flag should be absent; the plan output must be reviewed and explicitly approved by a responsible engineer before apply proceeds.

---

## Errors and Resolutions

No errors were encountered during this implementation. All commands executed cleanly in the following sequence:

| Step | Command | Result |
|---|---|---|
| Directory inspection | `ls -la` | Success |
| Provider review | `cat provider.tf` | Success |
| File authoring | `cat > main.tf << 'EOF'` | Success |
| Configuration verification | `cat main.tf` | Success |
| Initialisation | `terraform init` | Success (v5.91.0 installed) |
| Validation | `terraform validate` | `Success! The configuration is valid.` |
| Apply | `terraform apply -auto-approve` | `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` |
| State verification | `terraform state show aws_vpc.devops_vpc` | Full state output confirmed |

> In the event of an `Error: Failed to query available provider packages` during `terraform init`, verify that the Terraform registry endpoint is reachable from the host. In air-gapped or restricted environments, a Terraform mirror or a pre-downloaded provider binary in the plugin cache directory will be required.

> In the event of an `Error: error creating VPC` during apply, confirm that the LocalStack container is running and that the EC2 endpoint (`http://aws:4566`) is accessible from the Terraform host. Running `curl http://aws:4566` from the host is a reliable first diagnostic step.

---








<img width="1244" height="739" alt="image" src="https://github.com/user-attachments/assets/c7c57e53-6ae3-49c9-816d-e67d840d297c" />
<img width="1276" height="676" alt="image" src="https://github.com/user-attachments/assets/40679040-9fdd-4d81-9b86-fa9df13814e2" />
