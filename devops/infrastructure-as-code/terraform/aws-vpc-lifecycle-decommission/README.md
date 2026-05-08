# Terraform Managed VPC Decommission on AWS (LocalStack)

> Decommissioning a provisioned AWS VPC using Terraform's destroy workflow while preserving the Infrastructure-as-Code configuration for future re-provisioning.

---

## Table of Contents

- [Project Context](#project-context)
- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Architecture and Environment](#architecture-and-environment)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1 - Verify the Working Directory](#step-1---verify-the-working-directory)
  - [Step 2 - Inspect the Terraform Configuration](#step-2---inspect-the-terraform-configuration)
  - [Step 3 - Audit the Current State](#step-3---audit-the-current-state)
  - [Step 4 - Preview the Destruction Plan](#step-4---preview-the-destruction-plan)
  - [Step 5 - Execute the Destroy Operation](#step-5---execute-the-destroy-operation)
  - [Step 6 - Validate Post-Destroy State](#step-6---validate-post-destroy-state)
- [Configuration Reference](#configuration-reference)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Future Provisioning](#future-provisioning)

---

## Project Context

The Nautilus DevOps team is executing a phased migration of infrastructure to AWS. As part of this iterative approach, resources that were provisioned during earlier planning stages but are no longer operationally required are being systematically decommissioned to reduce cost and management overhead.

This document covers the targeted decommission of a VPC named **`datacenter-vpc`** in the **`us-east-1`** region, managed entirely through Terraform. The underlying Infrastructure-as-Code definition is intentionally preserved to allow reprovisioning at any future point without rework.

---

## Problem Statement

A VPC (`datacenter-vpc`) was provisioned in AWS `us-east-1` during an earlier phase of the cloud migration. After a review cycle, the team determined this VPC is no longer required in its current form. The resource must be deleted cleanly through Terraform to:

- Maintain state consistency between the Terraform state file and the actual infrastructure.
- Avoid orphaned resources that bypass IaC governance.
- Ensure the provisioning code remains intact for future use.

---

## Solution Overview

The decommission is performed using Terraform's native `destroy` command, which reads the existing state, generates a destruction execution plan, and tears down only the resources it manages. The `main.tf` configuration file is left untouched so the VPC can be reprovisioned at any time by running `terraform apply`.

---

## Architecture and Environment

| Property | Value |
|---|---|
| Cloud Provider | AWS (LocalStack emulation) |
| Region | `us-east-1` |
| Resource Type | AWS VPC |
| Resource Name | `datacenter-vpc` |
| CIDR Block | `10.0.0.0/16` |
| VPC ID | `vpc-97447fce31eb7e4ee` |
| Terraform Version | `1.11.0` |
| AWS Provider Version | `5.91.0` |
| State Backend | Local (`terraform.tfstate`) |
| IaC Working Directory | `/home/bob/terraform` |

The environment uses **LocalStack** running at `http://aws:4566` as the AWS API endpoint, simulating a full AWS environment locally. All service endpoints (EC2, IAM, S3, RDS, etc.) are routed through this address.

---

## Prerequisites

- Terraform `>= 1.11.0` installed and available on `$PATH`
- Access to the Terraform working directory at `/home/bob/terraform`
- LocalStack service running and reachable at `http://aws:4566`
- Read and write permissions on the `terraform.tfstate` file

---

## Repository Structure

```
/home/bob/terraform/
├── .terraform/                  # Provider plugins (auto-managed)
├── .terraform.lock.hcl          # Dependency lock file
├── main.tf                      # VPC resource definition
├── provider.tf                  # AWS provider and endpoint configuration
├── terraform.tfstate            # Local state file
└── README.MD                    # Original project notes
```

---

## Implementation Guide

### Step 1 - Verify the Working Directory

Navigate to the Terraform working directory and confirm all required configuration files are present before performing any operations.

```bash
ls -la
```

**Expected output confirms the presence of:**

- `main.tf` - resource definitions
- `provider.tf` - provider configuration
- `terraform.tfstate` - current infrastructure state
- `.terraform/` - initialised provider plugins
- `.terraform.lock.hcl` - provider version lock

*Screenshot: Directory listing showing all Terraform configuration files present*

<img width="1033" height="525" alt="image" src="https://github.com/user-attachments/assets/0c90f451-7107-4e64-bb45-7e8a19623899" />

---

### Step 2 - Inspect the Terraform Configuration

Review both configuration files to understand the resource being managed and the provider setup before making any changes.

**Inspect the resource definition:**

```bash
cat main.tf
```

```hcl
resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "datacenter-vpc"
  }
}
```

**Inspect the provider configuration:**

```bash
cat provider.tf
```

The provider configuration routes all AWS API calls to LocalStack at `http://aws:4566` and disables credential validation and account ID resolution, which is standard practice for local development and testing environments using LocalStack.

*Screenshot: Output of `cat main.tf` and `cat provider.tf` showing resource and provider definitions*

<img width="1075" height="707" alt="image" src="https://github.com/user-attachments/assets/687e964a-9ad8-4912-ad21-5379aa4e6104" />

---

### Step 3 - Audit the Current State

Before initiating any destructive operation, confirm what resources Terraform is currently tracking and validate the live resource attributes match the expected values.

**List all resources in state:**

```bash
terraform state list
```

Output:
```
aws_vpc.this
```

**Inspect the full resource state:**

```bash
terraform state show aws_vpc.this
```

This command reveals all attributes Terraform has recorded for the VPC, including the assigned VPC ID, associated network ACLs, route tables, security groups, DHCP options, DNS settings, and owner ID. Reviewing this output before destruction confirms there are no unexpected dependencies or attributes that could indicate sub-resources requiring separate cleanup.

Key attributes confirmed:

| Attribute | Value |
|---|---|
| `id` | `vpc-97447fce31eb7e4ee` |
| `cidr_block` | `10.0.0.0/16` |
| `enable_dns_support` | `true` |
| `enable_dns_hostnames` | `false` |
| `instance_tenancy` | `default` |
| `tags["Name"]` | `datacenter-vpc` |

*Screenshot: Output of `terraform state list` confirming `aws_vpc.this` is tracked*

<img width="1048" height="529" alt="image" src="https://github.com/user-attachments/assets/8c083d6e-51bd-47c8-b738-1476cbe6df1b" />

*Screenshot: Output of `terraform state show aws_vpc.this` displaying full resource attributes*

<img width="1049" height="623" alt="image" src="https://github.com/user-attachments/assets/a6fbb841-857a-4088-b3f2-8f120b641063" />

---

### Step 4 - Preview the Destruction Plan

Always generate and review a destruction plan before applying it. The `-destroy` flag produces a plan that shows exactly which resources will be removed and what their current values are, without making any changes to infrastructure.

```bash
terraform plan -destroy
```

Terraform refreshes the state by re-reading the live resource (`vpc-97447fce31eb7e4ee`) and generates the execution plan. The plan output confirms:

- **0 resources to add**
- **0 resources to change**
- **1 resource to destroy** (`aws_vpc.this`)

Every attribute set to be removed is listed with a `-> null` suffix, providing complete visibility into what will be deallocated.

> **Note:** The `-out` flag was not used here. In production pipelines, it is strongly recommended to save the plan with `-out=tfplan` and pass the saved plan file to `terraform apply tfplan` to guarantee that exactly what was reviewed gets applied.

*Screenshot: Output of `terraform plan -destroy` showing the planned destruction of `aws_vpc.this`*

---

### Step 5 - Execute the Destroy Operation

With the plan reviewed and confirmed, execute the destroy command. Terraform will present the same execution plan and require explicit confirmation before proceeding.

```bash
terraform destroy
```

Terraform re-runs the refresh, presents the destruction plan, and prompts for confirmation:

```
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

Enter `yes` to confirm. Terraform proceeds with the deletion:

```
aws_vpc.this: Destroying... [id=vpc-97447fce31eb7e4ee]
aws_vpc.this: Destruction complete after 0s

Destroy complete! Resources: 1 destroyed.
```

*Screenshot: `terraform destroy` confirmation prompt and successful destruction output*

---

### Step 6 - Validate Post-Destroy State

After the destroy operation completes, perform a final validation to confirm the state file is clean and no orphaned resources remain under Terraform management.

**Verify the state is empty:**

```bash
terraform state list
```

Output: *(empty - no managed resources)*

**Inspect the raw state file:**

```bash
cat terraform.tfstate
```

```json
{
  "version": 4,
  "terraform_version": "1.11.0",
  "serial": 3,
  "lineage": "b5bd801f-207e-a622-3490-cb49ac0c0b6b",
  "outputs": {},
  "resources": [],
  "check_results": null
}
```

The `resources` array is empty, confirming the VPC has been fully decommissioned and the Terraform state is in a clean, consistent condition.

**Confirm the provisioning code is intact:**

```bash
cat main.tf
```

The `main.tf` file remains unchanged with the full VPC resource definition preserved for future use.

*Screenshot: Empty output from `terraform state list` confirming no managed resources*

*Screenshot: `terraform.tfstate` showing empty `resources` array*

*Screenshot: `cat main.tf` confirming provisioning code is preserved and unmodified*

---

## Configuration Reference

### `main.tf`

```hcl
resource "aws_vpc" "this" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "datacenter-vpc"
  }
}
```

### `provider.tf`

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

## Best Practices Applied

**Always run `terraform plan` before `terraform apply` or `terraform destroy`**
Running `terraform plan -destroy` before executing the actual destroy provides a human-readable preview of every resource and attribute that will be removed. This is a mandatory step in any production workflow and prevents accidental deletion of resources not intended for removal.

**Use `terraform state show` before destructive operations**
Inspecting the live state of a resource before destroying it gives the operator full visibility into what is being removed. This is especially important in environments where multiple engineers may have made out-of-band changes.

**Preserve Infrastructure-as-Code definitions after decommission**
The `main.tf` file was left intact after the VPC was destroyed. This follows the IaC principle of treating configuration as a source of truth. If the VPC is needed again, it can be reprovisioned with a single `terraform apply` without any code changes.

**Validate state consistency post-destroy**
After any destructive operation, checking both `terraform state list` (managed resources) and inspecting the raw `terraform.tfstate` JSON confirms the state file accurately reflects the current infrastructure. An empty `resources` array with a correctly incremented `serial` number signals a clean state.

**Use explicit provider version pinning**
The `required_providers` block pins the AWS provider to `5.91.0` and the `.terraform.lock.hcl` file records the exact checksum. This prevents unexpected behaviour caused by provider version drift across team members or CI environments.

**Keep IaC working directories clean and well-structured**
All Terraform files are co-located in a single working directory with a clear separation between provider configuration (`provider.tf`), resource definitions (`main.tf`), and state management. This structure makes the codebase easy to audit and hand off.

---

## Lessons Learned

**Terraform destroy is safe when preceded by a plan review**
The two-phase approach (plan then apply/destroy) is not just a formality. The plan output explicitly shows every attribute being nullified, making it easy to catch cases where sub-resources or dependencies might block or partially fail the destruction. In this case, the VPC had no subnets, internet gateways, or route table associations, so destruction completed in 0 seconds cleanly.

**LocalStack endpoint configuration must be complete**
When working with LocalStack, every AWS service that Terraform may internally call during a `plan` or `destroy` operation (including STS for identity resolution and EC2 for the VPC itself) must have its endpoint explicitly overridden. Missing even one endpoint can cause confusing authentication or connectivity errors that appear unrelated to the resource being managed.

**`skip_credentials_validation` and `skip_requesting_account_id` are essential for LocalStack**
Without these flags, the AWS provider attempts to validate credentials and resolve the account ID via STS, which fails in a LocalStack environment. These settings are safe to use in isolated local and testing environments but must never be carried into configurations targeting real AWS accounts.

**State serial increments are a reliable integrity signal**
After each Terraform operation (apply or destroy), the `serial` field in `terraform.tfstate` increments. Checking this value before and after an operation confirms that Terraform successfully wrote the state update. A `serial` that does not increment may indicate a failed state write, which requires immediate investigation before any further operations.

**IaC separation of concerns reduces risk**
By separating provider configuration (`provider.tf`) from resource definitions (`main.tf`), it is possible to update endpoint configurations, credentials, or regions without touching resource code. This also makes code review and auditing simpler in team environments.

---

## Future Provisioning

When the `datacenter-vpc` needs to be reprovisioned, the existing `main.tf` configuration is fully ready. No code changes are required. From the working directory, run:

```bash
terraform plan
terraform apply
```

Terraform will read the `main.tf` definition, generate a plan to create the VPC with CIDR `10.0.0.0/16` and the `datacenter-vpc` name tag, and provision it against the configured LocalStack endpoint. The state file will be updated automatically upon successful apply.





<img width="1075" height="692" alt="image" src="https://github.com/user-attachments/assets/f451cc8d-ef09-42f7-a223-7ac9f0c23023" />
<img width="1073" height="782" alt="image" src="https://github.com/user-attachments/assets/bf235a0c-6ba2-472c-887c-8e0854f30deb" />
<img width="1047" height="262" alt="image" src="https://github.com/user-attachments/assets/238d2dc7-6c44-42ca-91aa-661e3f839ec3" />
<img width="1051" height="439" alt="image" src="https://github.com/user-attachments/assets/c53fb175-6f79-4ef5-8138-3b8c845309b8" />
<img width="1050" height="600" alt="image" src="https://github.com/user-attachments/assets/a98b1dab-5425-4e02-a3ee-29008beb882a" />
