# Terraform AWS Elastic IP Provisioning on LocalStack

> Provisioning an AWS Elastic IP (EIP) address using Terraform Infrastructure-as-Code against a LocalStack endpoint, with configuration values managed through a dedicated variables file.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Inspect the Working Directory](#step-1-inspect-the-working-directory)
  * [Step 2: Review the Provider Configuration](#step-2-review-the-provider-configuration)
  * [Step 3: Create the Variables File](#step-3-create-the-variables-file)
  * [Step 4: Create the Main Terraform Configuration](#step-4-create-the-main-terraform-configuration)
  * [Step 5: Verify the Configuration Files](#step-5-verify-the-configuration-files)
  * [Step 6: Initialize the Terraform Working Directory](#step-6-initialize-the-terraform-working-directory)
  * [Step 7: Validate the Configuration](#step-7-validate-the-configuration)
  * [Step 8: Generate and Review the Execution Plan](#step-8-generate-and-review-the-execution-plan)
  * [Step 9: Apply the Configuration](#step-9-apply-the-configuration)
  * [Step 10: Verify the Deployed Resource](#step-10-verify-the-deployed-resource)
* [Resource Verification](#resource-verification)
* [Best Practices Applied](#best-practices-applied)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Project Overview

The Nautilus DevOps team is executing a phased migration of their infrastructure to the AWS cloud. As part of this initiative, the team requires an Elastic IP (EIP) address to support stable, externally routable access for specific workloads transitioning to the VPC environment.

This implementation provisions a single AWS Elastic IP address using Terraform, targeting a LocalStack-simulated AWS environment. The EIP name tag is managed through a Terraform input variable to enforce clean separation between configuration values and resource definitions.

**Key Requirement:** The Elastic IP must be created with the name tag `xfusion-eip`, stored in a variable named `KKE_eip`, with the resource definition in `main.tf` referencing the value from `variables.tf`.

---

## Architecture and Design Intent

```
/home/bob/terraform/
    provider.tf      # AWS provider configuration pointing to LocalStack endpoints
    variables.tf     # Input variable declarations (KKE_eip = "xfusion-eip")
    main.tf          # Resource definition referencing the variable
```

**Infrastructure Flow:**

```
Terraform CLI
    |
    v
provider.tf (hashicorp/aws 5.91.0, LocalStack endpoint http://aws:4566)
    |
    v
variables.tf (KKE_eip = "xfusion-eip")
    |
    v
main.tf (aws_eip resource, domain = "vpc", Name tag = var.KKE_eip)
    |
    v
LocalStack (simulated AWS EC2/VPC service at http://aws:4566)
    |
    v
aws_eip.eip (allocation_id: eipalloc-94039e2238b43c4c8, public_ip: 127.231.20.38)
```

The `domain = "vpc"` attribute allocates the EIP to the VPC scope (as opposed to EC2-Classic, which is deprecated). The design intentionally keeps the name value in `variables.tf` to make it trivially overridable at plan or apply time without modifying the resource definition.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Terraform | v1.x or later |
| AWS Provider | hashicorp/aws v5.91.0 |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |
| Shell Access | Terminal opened in the Terraform working directory |

---

## Repository Structure

```
terraform/
    provider.tf          # Provider block with LocalStack endpoint overrides
    variables.tf         # Input variable: KKE_eip (default: "xfusion-eip")
    main.tf              # aws_eip resource definition
    .terraform/          # Provider plugin cache (generated after init)
    .terraform.lock.hcl  # Dependency lock file (generated after init)
    terraform.tfstate    # State file (generated after apply)
```

---

## Implementation Guide

### Step 1: Inspect the Working Directory

Before creating any new files, confirm the existing state of the working directory. The directory was pre-populated with `provider.tf` and a `README.MD`.

```bash
ls -la
```

**Output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 May 13 06:09 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> Screenshot: Working directory listing showing `provider.tf` and `README.MD` as the only pre-existing files

<img width="523" height="319" alt="image" src="https://github.com/user-attachments/assets/6a8b852a-802d-45b0-b700-89559978ff5e" />

---

### Step 2: Review the Provider Configuration

Examine the existing `provider.tf` to understand the AWS provider version, region, and all LocalStack endpoint overrides in scope.

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

> Screenshot: Terminal output of `cat provider.tf` showing all LocalStack endpoint overrides

<img width="537" height="376" alt="image" src="https://github.com/user-attachments/assets/e563000b-9e1d-4348-ae8e-0b8e3063d4e5" />

**Key observations:**
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack to bypass real AWS credential checks
* All service endpoints, including `ec2` (which handles EIP allocation), are routed to `http://aws:4566`
* The provider version is pinned to `5.91.0` for reproducibility

---

### Step 3: Create the Variables File

Create `variables.tf` to declare the `KKE_eip` input variable with the default value `xfusion-eip`. Using a heredoc ensures the file is written atomically in a single shell operation.

```bash
cat > variables.tf << 'EOF'
variable "KKE_eip" {
  description = "Name tag for the AWS Elastic IP address"
  type        = string
  default     = "xfusion-eip"
}
EOF
```

Confirm the file was written correctly:

```bash
cat variables.tf
```

**Output:**

```hcl
variable "KKE_eip" {
  description = "Name tag for the AWS Elastic IP address"
  type        = string
  default     = "xfusion-eip"
}
```

> Screenshot: Terminal output of `cat variables.tf` confirming variable declaration with correct name, type, and default value

<img width="523" height="366" alt="image" src="https://github.com/user-attachments/assets/65d5853b-131f-4142-94b1-1b1a070be345" />

---

### Step 4: Create the Main Terraform Configuration

Create `main.tf` to define the `aws_eip` resource. The resource references `var.KKE_eip` for the Name tag, keeping the resource definition decoupled from the literal value.

```bash
cat > main.tf << 'EOF'
resource "aws_eip" "eip" {
  domain = "vpc"

  tags = {
    Name = var.KKE_eip
  }
}
EOF
```

Confirm the file was written correctly:

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_eip" "eip" {
  domain = "vpc"

  tags = {
    Name = var.KKE_eip
  }
}
```

> Screenshot: Terminal output of `cat main.tf` confirming the resource block with `domain = "vpc"` and `Name = var.KKE_eip`

<img width="525" height="319" alt="image" src="https://github.com/user-attachments/assets/cd437d2f-f34a-4c5b-aafe-610015f686a2" />

---

### Step 5: Verify the Configuration Files

After creating both files, confirm the directory now contains all required configuration files before proceeding with initialization.

```bash
ls -la
```

**Output:**

```
total 28
drwxr-xr-x 1 bob bob 4096 May 13 06:17 .
drwxr-x--- 1 bob bob 4096 May 13 06:09 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob   85 May 13 06:17 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-r--r-- 1 bob bob  134 May 13 06:16 variables.tf
```

> Screenshot: Updated directory listing confirming `main.tf` and `variables.tf` are present alongside the pre-existing `provider.tf`

<img width="524" height="328" alt="image" src="https://github.com/user-attachments/assets/49102f28-b758-4cdd-9804-009d7453705f" />

All three configuration files are in place. The working directory is ready for initialization.

---

### Step 6: Initialize the Terraform Working Directory

Run `terraform init` to download the required provider plugin and create the dependency lock file.

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

> Screenshot: Terminal output of `terraform init` showing successful provider installation and lock file creation

<img width="523" height="281" alt="image" src="https://github.com/user-attachments/assets/9733510e-ccc4-4efe-8a2a-03dc5468b64f" />

The lock file `.terraform.lock.hcl` pins the provider to `v5.91.0` for all future runs in this workspace.

---

### Step 7: Validate the Configuration

Run `terraform validate` to perform a static syntax and semantic check of all `.tf` files without making any API calls.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

> Screenshot: Terminal output of `terraform validate` showing `Success! The configuration is valid.`

<img width="524" height="241" alt="image" src="https://github.com/user-attachments/assets/3178c19c-dd67-4a40-b5d0-74aa4fe32c44" />

Validation confirms there are no syntax errors, undefined variable references, or type mismatches in the configuration.

---

### Step 8: Generate and Review the Execution Plan

Run `terraform plan` to preview the exact actions Terraform will take. Reviewing the plan before applying is a mandatory production discipline.

```bash
terraform plan
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_eip.eip will be created
  + resource "aws_eip" "eip" {
      + allocation_id        = (known after apply)
      + arn                  = (known after apply)
      + domain               = "vpc"
      + id                   = (known after apply)
      + public_dns           = (known after apply)
      + public_ip            = (known after apply)
      + tags                 = {
          + "Name" = "xfusion-eip"
        }
      + tags_all             = {
          + "Name" = "xfusion-eip"
        }
      + vpc                  = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> Screenshot: Full `terraform plan` output showing the `aws_eip.eip` resource to be created with `Name = "xfusion-eip"` and `domain = "vpc"`

The plan shows exactly one resource addition. The `Name` tag is correctly resolved to `"xfusion-eip"` from the variable default, confirming the variable reference is functioning as intended.

---

### Step 9: Apply the Configuration

Apply the configuration with the `-auto-approve` flag to provision the Elastic IP without an interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
...
Plan: 1 to add, 0 to change, 0 to destroy.
aws_eip.eip: Creating...
aws_eip.eip: Creation complete after 0s [id=eipalloc-94039e2238b43c4c8]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> Screenshot: Terminal output of `terraform apply -auto-approve` showing `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` with allocation ID `eipalloc-94039e2238b43c4c8`

The resource was created in under one second. The allocation ID `eipalloc-94039e2238b43c4c8` is now the stable identifier for this Elastic IP in the LocalStack state.

---

### Step 10: Verify the Deployed Resource

#### List Resources in State

Confirm the resource is tracked in Terraform state.

```bash
terraform state list
```

**Output:**

```
aws_eip.eip
```

> Screenshot: Output of `terraform state list` showing `aws_eip.eip` as the single managed resource

#### Inspect Full Resource State

Retrieve the complete attribute set of the provisioned EIP from the state file.

```bash
terraform state show aws_eip.eip
```

**Output:**

```hcl
# aws_eip.eip:
resource "aws_eip" "eip" {
    allocation_id            = "eipalloc-94039e2238b43c4c8"
    arn                      = "arn:aws:ec2:us-east-1::elastic-ip/eipalloc-94039e2238b43c4c8"
    association_id           = null
    carrier_ip               = null
    customer_owned_ip        = null
    customer_owned_ipv4_pool = null
    domain                   = "vpc"
    id                       = "eipalloc-94039e2238b43c4c8"
    instance                 = null
    network_border_group     = null
    network_interface        = null
    private_ip               = null
    ptr_record               = null
    public_dns               = "ec2-127-231-20-38.compute-1.amazonaws.com"
    public_ip                = "127.231.20.38"
    public_ipv4_pool         = null
    tags                     = {
        "Name" = "xfusion-eip"
    }
    tags_all                 = {
        "Name" = "xfusion-eip"
    }
    vpc                      = true
}
```

> Screenshot: Full output of `terraform state show aws_eip.eip` confirming `allocation_id`, `public_ip = "127.231.20.38"`, `domain = "vpc"`, and `Name = "xfusion-eip"`

---

## Resource Verification

| Attribute | Expected Value | Observed Value |
|---|---|---|
| Resource address | `aws_eip.eip` | `aws_eip.eip` |
| Name tag | `xfusion-eip` | `xfusion-eip` |
| Domain | `vpc` | `vpc` |
| Allocation ID | (assigned by provider) | `eipalloc-94039e2238b43c4c8` |
| Public IP | (assigned by provider) | `127.231.20.38` |
| VPC scope | `true` | `true` |
| Association | None (unattached) | `null` |

All attributes match the expected specification. The EIP is allocated, unattached, and ready for association with a network interface or instance.

---

## Best Practices Applied

* **Variable-driven configuration:** The EIP name tag is stored in `variables.tf` as `KKE_eip` rather than hardcoded in `main.tf`. This allows the tag value to be overridden at runtime using `-var` flags or `.tfvars` files without touching the resource definition.

* **Explicit VPC domain declaration:** Setting `domain = "vpc"` is explicit and forward-compatible. The legacy EC2-Classic `domain = "standard"` is deprecated, and explicitly setting the domain prevents ambiguity across provider versions.

* **Provider version pinning:** The `required_providers` block locks the AWS provider to `v5.91.0`, ensuring consistent behavior across all team members and CI/CD pipelines. The `.terraform.lock.hcl` file reinforces this at the dependency resolution layer.

* **Plan before apply:** Running `terraform plan` before `terraform apply` ensures the operator reviews the exact diff before committing changes. In production workflows, the plan output should be saved with `-out=tfplan` and passed to `terraform apply tfplan` to guarantee the applied plan matches the reviewed one.

* **State verification post-apply:** Running `terraform state list` and `terraform state show` immediately after apply confirms the resource was written to state correctly and all expected attributes are present. This step is especially important in LocalStack environments where API behavior can differ subtly from real AWS.

* **Separation of concerns across files:** Keeping provider configuration, variable declarations, and resource definitions in separate files (`provider.tf`, `variables.tf`, `main.tf`) follows standard Terraform module conventions, improving readability and making targeted edits safer.

* **Heredoc for file creation:** Using `cat > filename << 'EOF' ... EOF` for file creation ensures the content is written atomically in a single shell operation, preventing partial writes and making the commands reproducible in automated provisioning scripts.

---

## Errors and Resolutions

No errors were encountered during this implementation. The following scenarios represent common failure modes for this pattern and how to resolve them.

---

**Potential Issue: `domain` attribute deprecated warning**

* **Symptom:** Terraform emits a deprecation warning about the `vpc` attribute being set implicitly alongside the `domain` argument.
* **Root Cause:** In AWS provider versions prior to 5.x, the `vpc = true` argument was used instead of `domain = "vpc"`. Some older module references use both simultaneously.
* **Resolution:** Use only `domain = "vpc"` with AWS provider v5.x. Do not set both `domain` and `vpc` in the same resource block.

---

**Potential Issue: LocalStack endpoint connection refused**

* **Symptom:** `terraform apply` fails with `connection refused` or `no such host` targeting `http://aws:4566`.
* **Root Cause:** LocalStack is not running, or the `aws` hostname is not resolvable from the Terraform execution context.
* **Resolution:** Confirm LocalStack is running with `curl http://aws:4566/_localstack/health`. If the hostname does not resolve, verify `/etc/hosts` or the Docker network configuration linking the `aws` hostname to the LocalStack container.

---

**Potential Issue: State file conflict after re-apply**

* **Symptom:** Running `terraform apply` a second time attempts to create a duplicate resource.
* **Root Cause:** LocalStack state is ephemeral by default. If LocalStack restarted between runs, the real EIP allocation no longer exists but Terraform state still references it, causing a drift.
* **Resolution:** Run `terraform destroy -auto-approve` to clean the state, then re-apply. Alternatively, use `terraform import` to reconcile if the resource was recreated with a known ID.

---

## Lessons Learned

* **`domain = "vpc"` is the correct modern argument.** In AWS provider v5.x, `domain = "vpc"` replaces the older `vpc = true` boolean. Using the boolean causes a deprecation warning and may be removed in future provider releases. Always use the string-form `domain` argument for new configurations.

* **Variable separation pays dividends at scale.** Storing the EIP name in a variable instead of a hardcoded tag value may seem like overhead for a single resource, but this pattern scales cleanly. When the same pattern is applied across dozens of resources, a single `.tfvars` file or CI/CD variable injection can configure an entire environment without touching any resource files.

* **`terraform validate` catches variable reference errors early.** If `main.tf` had referenced a variable name that did not exist in `variables.tf`, `terraform validate` would have caught it before any API call was made. Building validate into a pre-commit hook or CI pipeline gate eliminates an entire class of apply-time failures.

* **`terraform state show` is more informative than `terraform output`.** For resources without explicit `output` blocks, `terraform state show` exposes the full attribute map directly from state. This is the fastest way to confirm post-apply attribute values without adding output blocks to the configuration.

* **LocalStack allocates syntactically valid but non-routable IPs.** The `public_ip = "127.231.20.38"` returned by LocalStack follows the EIP format but is a loopback-range address. This is expected behavior and does not indicate a problem. When migrating this configuration to a real AWS account, the public IP will be drawn from Amazon's owned IP space.

---

## References

* [Terraform AWS Provider: aws_eip Resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip)
* [Terraform Input Variables](https://developer.hashicorp.com/terraform/language/values/variables)
* [LocalStack AWS Simulation](https://docs.localstack.cloud/overview/)
* [Terraform CLI: plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
* [Terraform CLI: state show](https://developer.hashicorp.com/terraform/cli/commands/state/show)
* [AWS Elastic IP Addresses Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)









<img width="537" height="341" alt="image" src="https://github.com/user-attachments/assets/cdd22ac8-04a8-46e1-ae6f-c3638f890676" />
<img width="537" height="355" alt="image" src="https://github.com/user-attachments/assets/3c156cc2-3fcd-4cb0-96bd-f709598f7d45" />
<img width="525" height="347" alt="image" src="https://github.com/user-attachments/assets/5dc47ab8-a2e0-4266-bf8d-e94d92416470" />
<img width="523" height="346" alt="image" src="https://github.com/user-attachments/assets/428beac9-dae3-4c3b-b1ad-e501ecffe1e7" />



