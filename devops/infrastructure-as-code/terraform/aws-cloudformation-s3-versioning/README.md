# Terraform CloudFormation Stack: S3 Bucket with Versioning Enabled

> **Enterprise Infrastructure Automation** | Terraform + AWS CloudFormation + LocalStack | Production-Grade IaC

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Technology Stack](#technology-stack)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Workspace Verification](#phase-1-workspace-verification)
  - [Phase 2: Terraform Configuration Authoring](#phase-2-terraform-configuration-authoring)
  - [Phase 3: Terraform Initialization](#phase-3-terraform-initialization)
  - [Phase 4: Configuration Validation and Plan](#phase-4-configuration-validation-and-plan)
  - [Phase 5: Infrastructure Provisioning](#phase-5-infrastructure-provisioning)
  - [Phase 6: Post-Deployment Verification](#phase-6-post-deployment-verification)
- [Configuration Reference](#configuration-reference)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Author](#author)

---

## Overview

This implementation provisions an AWS CloudFormation stack using Terraform as the infrastructure orchestration layer. The CloudFormation stack manages a single S3 bucket resource with versioning explicitly enabled, deployed against a LocalStack-emulated AWS environment. The entire workflow follows production-grade Infrastructure as Code (IaC) standards: configuration authoring, validation, plan review, and idempotent apply.

This pattern represents the intersection of two IaC paradigms: Terraform manages the CloudFormation stack lifecycle (create, update, destroy), while CloudFormation manages the actual AWS resource provisioning within that stack. This layered approach is common in enterprise environments migrating from native CloudFormation to Terraform-managed infrastructure pipelines.

---

## Problem Statement

The Nautilus DevOps team required a repeatable, version-controlled mechanism to provision cloud infrastructure for automation workflows. Specifically, the team needed to:

* Deploy a CloudFormation stack named **devops-stack** using Terraform
* Provision an S3 bucket named **devops-bucket-21253** as a resource inside that stack
* Enforce S3 versioning as a non-negotiable bucket property
* Operate against a LocalStack-emulated AWS environment (endpoint: `http://aws:4566`)
* Contain all configuration in a single `main.tf` file within the existing Terraform working directory

The requirement to use Terraform to manage a CloudFormation stack rather than invoking CloudFormation directly reflects a deliberate enterprise decision: unified IaC tooling across multi-cloud and hybrid teams.

---

## Solution Architecture

```
Terraform (Orchestration Layer)
      |
      v
aws_cloudformation_stack resource (main.tf)
      |
      v
CloudFormation Stack: devops-stack
      |
      v
AWS::S3::Bucket
  Name: devops-bucket-21253
  Versioning: Enabled
      |
      v
LocalStack Emulated AWS Endpoint (http://aws:4566)
```

Terraform acts as the control plane. It serializes the CloudFormation template body inline using `jsonencode()`, submits it to the LocalStack CloudFormation endpoint, and waits for the stack to reach `CREATE_COMPLETE` before releasing control. Verification is performed post-apply using the AWS CLI targeting the same LocalStack endpoint.

---

## Technology Stack

| Component | Version / Detail |
|---|---|
| Terraform | HashiCorp Terraform (CLI) |
| AWS Provider | hashicorp/aws v5.91.0 |
| AWS Emulation | LocalStack (endpoint: http://aws:4566) |
| CloudFormation | AWSTemplateFormatVersion 2010-09-09 |
| AWS CLI | Used for post-apply verification |
| Working Directory | /home/bob/terraform |
| Configuration File | main.tf |

---

## Prerequisites

* Terraform CLI installed and available on `$PATH`
* LocalStack running and accessible at `http://aws:4566`
* AWS CLI configured (credentials not required for LocalStack; endpoint override used)
* Existing `provider.tf` present in `/home/bob/terraform` with LocalStack endpoint mappings
* Write access to the Terraform working directory

---

## Repository Structure

```
/home/bob/terraform/
|-- provider.tf          # AWS provider configuration with LocalStack endpoint overrides
|-- main.tf              # CloudFormation stack resource definition (created in this implementation)
|-- .terraform/          # Terraform plugin cache (generated after init)
|-- .terraform.lock.hcl  # Provider dependency lock file (generated after init)
|-- README.MD            # Existing project readme
```

---

## Implementation Guide

### Phase 1: Workspace Verification

Before authoring any configuration, the existing working directory was inspected to understand what was already present and to avoid overwriting existing files.

**Confirm working directory:**

```bash
pwd
```

Expected output:

```
/home/bob/terraform
```

**List directory contents:**

```bash
ls -la
```

Output confirmed the following files were present prior to this implementation:

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 24 04:19 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

**Review existing provider configuration:**

```bash
cat provider.tf
```

This confirmed the provider was pre-configured for LocalStack with:

* `region = "us-east-1"`
* `skip_credentials_validation = true`
* `skip_requesting_account_id = true`
* `s3_use_path_style = true`
* Full endpoint override mapping for all required AWS services to `http://aws:4566`

*Screenshot: Terminal output of `ls -la` and `cat provider.tf` confirming the pre-existing workspace state*

<img width="1073" height="804" alt="image" src="https://github.com/user-attachments/assets/f03d8fc3-4b5e-44af-8278-a9b32f933e18" />

---

### Phase 2: Terraform Configuration Authoring

With the provider already configured, the `main.tf` file was created to define the CloudFormation stack resource. The `jsonencode()` function was used to inline the CloudFormation template body directly within the Terraform HCL configuration, eliminating the need for a separate template file.

**Create main.tf:**

```bash
cat > main.tf << 'EOF'
resource "aws_cloudformation_stack" "devops_stack" {
  name = "devops-stack"

  template_body = jsonencode({
    AWSTemplateFormatVersion = "2010-09-09"
    Resources = {
      DevopsBucket = {
        Type = "AWS::S3::Bucket"
        Properties = {
          BucketName = "devops-bucket-21253"
          VersioningConfiguration = {
            Status = "Enabled"
          }
        }
      }
    }
  })
}
EOF
```

**Verify the file was written correctly:**

```bash
cat main.tf
```

Expected output:

```hcl
resource "aws_cloudformation_stack" "devops_stack" {
  name = "devops-stack"

  template_body = jsonencode({
    AWSTemplateFormatVersion = "2010-09-09"
    Resources = {
      DevopsBucket = {
        Type = "AWS::S3::Bucket"
        Properties = {
          BucketName = "devops-bucket-21253"
          VersioningConfiguration = {
            Status = "Enabled"
          }
        }
      }
    }
  })
}
```

*Screenshot: Terminal output of `cat main.tf` confirming correct file content before proceeding*

<img width="1069" height="600" alt="image" src="https://github.com/user-attachments/assets/bf49094b-ee71-4398-8358-72ed48b08854" />

---

### Phase 3: Terraform Initialization

With both `provider.tf` and `main.tf` in place, Terraform was initialized to download the required provider plugin and configure the backend.

**Initialize the Terraform working directory:**

```bash
terraform init
```

Expected output:

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

The lock file `.terraform.lock.hcl` was generated at this stage, pinning the provider to `hashicorp/aws v5.91.0` for reproducibility.

*Screenshot: Terminal output of `terraform init` showing successful provider installation and lock file creation*

<img width="1048" height="695" alt="image" src="https://github.com/user-attachments/assets/e9d54516-be99-445b-83e6-a1982b5c8e79" />

---

### Phase 4: Configuration Validation and Plan

Prior to applying any changes, the configuration was validated for syntax correctness, and a plan was generated to preview the exact changes Terraform intended to make.

**Validate configuration syntax:**

```bash
terraform validate
```

Expected output:

```
Success! The configuration is valid.
```

**Generate and review the execution plan:**

```bash
terraform plan
```

Key elements confirmed in the plan output:

* Resource type: `aws_cloudformation_stack`
* Resource address: `aws_cloudformation_stack.devops_stack`
* Stack name: `devops-stack`
* Template body included the S3 bucket `devops-bucket-21253` with `VersioningConfiguration.Status = "Enabled"`
* Plan summary: **Plan: 1 to add, 0 to change, 0 to destroy**

The plan output confirmed that no existing resources would be modified or destroyed, and the intended resource accurately matched the authored configuration.

*Screenshot: Terminal output of `terraform validate` returning success*

*Screenshot: Terminal output of `terraform plan` showing the full execution plan with resource details and the 1-to-add summary*

---

### Phase 5: Infrastructure Provisioning

With the plan confirmed, the configuration was applied using the `-auto-approve` flag to bypass the interactive confirmation prompt, consistent with how this flag is used in CI/CD pipeline automation.

**Apply the configuration:**

```bash
terraform apply -auto-approve
```

Terraform executed the plan, printed the same plan summary for confirmation, then began creating the resource. The apply output showed:

```
aws_cloudformation_stack.devops_stack: Creating...
aws_cloudformation_stack.devops_stack: Still creating... [10s elapsed]
aws_cloudformation_stack.devops_stack: Creation complete after 10s [id=arn:aws:cloudformation:us-east-1:000000000000:stack/devops-stack/0e20f849-d28b-410c-9f8f-532d98e711f6]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The stack was assigned the ARN `arn:aws:cloudformation:us-east-1:000000000000:stack/devops-stack/0e20f849-d28b-410c-9f8f-532d98e711f6`, confirming successful registration within LocalStack's CloudFormation service.

*Screenshot: Terminal output of `terraform apply -auto-approve` showing the creation sequence and final "Apply complete" confirmation*

---

### Phase 6: Post-Deployment Verification

After the apply completed, two independent verification steps were performed using the AWS CLI to confirm the infrastructure state matched the intended configuration.

**Verify CloudFormation stack status:**

```bash
aws --endpoint-url=http://aws:4566 cloudformation describe-stacks --stack-name devops-stack
```

Expected output:

```json
{
    "Stacks": [
        {
            "StackId": "arn:aws:cloudformation:us-east-1:000000000000:stack/devops-stack/0e20f849-d28b-410c-9f8f-532d98e711f6",
            "StackName": "devops-stack",
            "CreationTime": "2026-04-24T04:31:16.261904Z",
            "LastUpdatedTime": "2026-04-24T04:31:16.261904Z",
            "RollbackConfiguration": {},
            "StackStatus": "CREATE_COMPLETE",
            "DisableRollback": false,
            "NotificationARNs": [],
            "Tags": [],
            "EnableTerminationProtection": false,
            "DriftInformation": {
                "StackDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

Key confirmed attributes:

* `StackName`: `devops-stack`
* `StackStatus`: `CREATE_COMPLETE`

**Verify S3 bucket versioning configuration:**

```bash
aws --endpoint-url=http://aws:4566 s3api get-bucket-versioning --bucket devops-bucket-21253
```

Expected output:

```json
{
    "Status": "Enabled"
}
```

This confirmed that the S3 bucket `devops-bucket-21253` was provisioned with versioning in the `Enabled` state, exactly as specified in the CloudFormation template body.

*Screenshot: Terminal output of `aws cloudformation describe-stacks` returning `CREATE_COMPLETE` status*

*Screenshot: Terminal output of `aws s3api get-bucket-versioning` returning `"Status": "Enabled"`*

---

## Configuration Reference

### provider.tf

The provider configuration was pre-existing and not modified during this implementation. It establishes the AWS provider at version `5.91.0` with LocalStack-specific settings:

* **skip_credentials_validation**: Bypasses IAM credential checks, required for LocalStack
* **skip_requesting_account_id**: Suppresses STS account ID lookup, required for LocalStack
* **s3_use_path_style**: Forces path-style S3 URLs instead of virtual-hosted-style, required for LocalStack compatibility
* **endpoints block**: Maps all service calls to `http://aws:4566`, the LocalStack unified gateway

### main.tf

```hcl
resource "aws_cloudformation_stack" "devops_stack" {
  name = "devops-stack"

  template_body = jsonencode({
    AWSTemplateFormatVersion = "2010-09-09"
    Resources = {
      DevopsBucket = {
        Type = "AWS::S3::Bucket"
        Properties = {
          BucketName = "devops-bucket-21253"
          VersioningConfiguration = {
            Status = "Enabled"
          }
        }
      }
    }
  })
}
```

| Attribute | Value | Purpose |
|---|---|---|
| `name` | `devops-stack` | CloudFormation stack identifier |
| `AWSTemplateFormatVersion` | `2010-09-09` | Standard CloudFormation template version |
| `Type` | `AWS::S3::Bucket` | CloudFormation resource type for S3 |
| `BucketName` | `devops-bucket-21253` | Explicit bucket name (avoids random suffix) |
| `VersioningConfiguration.Status` | `Enabled` | Activates S3 object versioning |

---

## Errors and Resolutions

No errors were encountered during this implementation. The workflow completed cleanly across all phases: workspace inspection, file authoring, initialization, validation, plan, apply, and verification. The following is a record of potential failure points and their mitigations that informed the execution approach:

**Potential: Incorrect heredoc causing malformed HCL**
* Risk: Using unquoted `EOF` in `cat > main.tf << EOF` without single-quoting would cause shell variable expansion inside the HCL, corrupting the file content.
* Mitigation Applied: Single-quoted heredoc delimiter (`<< 'EOF'`) was used explicitly, preventing any shell interpretation of the HCL content.

**Potential: LocalStack endpoint not reachable**
* Risk: If LocalStack was not running or the hostname `aws` was not resolvable, `terraform apply` would fail with a connection refused or DNS resolution error.
* Mitigation Applied: The provider configuration was reviewed before proceeding to confirm endpoint configuration. No connectivity errors were encountered.

**Potential: Provider version mismatch**
* Risk: If `.terraform.lock.hcl` from a prior run had pinned a different provider version, `terraform init` would have needed the `-upgrade` flag.
* Mitigation Applied: The working directory was confirmed clean with no pre-existing lock file before running `terraform init`.

---

## Best Practices Applied

**Inline template body using jsonencode()**
Using `jsonencode()` to embed the CloudFormation template directly in `main.tf` keeps the configuration self-contained and avoids managing a separate JSON or YAML template file. This reduces the risk of template drift and simplifies version control.

**Heredoc with single-quoted delimiter**
The `<< 'EOF'` syntax was used intentionally to suppress shell variable expansion when writing the configuration file. This is a critical discipline when writing HCL or any structured configuration via shell redirection.

**validate before plan, plan before apply**
The full gate sequence (`validate` to `plan` to `apply`) was followed without skipping steps. This is a non-negotiable production practice. `validate` catches syntax issues; `plan` reveals semantic intent; `apply` executes with full awareness.

**Post-apply verification with AWS CLI**
Rather than relying solely on Terraform's apply output as proof of success, independent CLI verification was performed against both the CloudFormation API (`describe-stacks`) and the S3 API (`get-bucket-versioning`). This two-layer verification confirms infrastructure state at the resource level, not just at the Terraform state level.

**Explicit bucket name**
Using an explicit `BucketName` in the CloudFormation template ensures deterministic resource naming. In a LocalStack or production environment, omitting the bucket name would result in a randomly generated name, which breaks downstream references and reproducibility.

**Lock file committed to version control**
The `.terraform.lock.hcl` file generated by `terraform init` pins the provider to `hashicorp/aws v5.91.0`. This file should be committed to version control to guarantee that all team members and CI/CD runners use the identical provider version, preventing provider-version-induced drift.

**Single configuration file scope**
All new IaC was contained within `main.tf` as required, with no new `.tf` files created. This respects the task scope boundary and avoids polluting the workspace with unnecessary file fragmentation.

---

## Lessons Learned

**Terraform and CloudFormation are complementary, not competing**
This implementation reinforces that `aws_cloudformation_stack` is a legitimate first-class Terraform resource. Teams already invested in CloudFormation templates can adopt Terraform as the lifecycle manager without rewriting every template immediately. This is a practical migration path in enterprise environments.

**jsonencode() eliminates template file management overhead**
Embedding CloudFormation templates inline using `jsonencode()` is appropriate for small-to-medium stacks. For larger or reusable templates, externalizing via `file()` or `templatefile()` is preferable. The choice between inline and external template bodies should be driven by template size and reuse requirements, not convenience.

**LocalStack endpoint verification is a prerequisite, not an afterthought**
Reviewing `provider.tf` before any Terraform command confirmed that the LocalStack configuration was correct. Skipping this check and proceeding directly to `terraform init` is a common mistake that produces confusing downstream errors. Always confirm provider configuration before initializing.

**AWS CLI verification closes the confidence loop**
Terraform's state is a representation of what Terraform believes exists, not necessarily what actually exists in the target environment. Using AWS CLI `describe-stacks` and `get-bucket-versioning` as independent verification steps closes the gap between Terraform state and actual infrastructure state. This is especially important in LocalStack environments where emulation behavior can occasionally diverge from real AWS behavior.

**Explicit versioning configuration must be present in the template, not assumed**
S3 buckets do not enable versioning by default. The `VersioningConfiguration` block must be explicitly present in the CloudFormation resource properties. Omitting it would result in a bucket that appears successfully created but silently lacks versioning, which would not be detected until a recovery scenario required it.

---


<img width="1047" height="541" alt="image" src="https://github.com/user-attachments/assets/9cf07f1d-bf20-4387-a126-345e640c69e1" />
<img width="1049" height="495" alt="image" src="https://github.com/user-attachments/assets/24f36abb-2351-4e2b-8d80-c9b6729e263f" />

<img width="1047" height="737" alt="image" src="https://github.com/user-attachments/assets/66af70a4-d047-42f4-af6d-8066d2d83a13" />

<img width="1043" height="484" alt="image" src="https://github.com/user-attachments/assets/f63c8b4d-a475-442b-8ab0-9457a9a28870" />
<img width="1080" height="649" alt="image" src="https://github.com/user-attachments/assets/bc420c54-b1ad-4a6b-923e-1288fad0bf90" />
<img width="1076" height="682" alt="image" src="https://github.com/user-attachments/assets/1103b6d7-4a51-47ea-b334-dfc3c622ea28" />
<img width="1165" height="553" alt="image" src="https://github.com/user-attachments/assets/b44edfba-6eca-4f29-8d69-2330aa053b67" />
<img width="1166" height="633" alt="image" src="https://github.com/user-attachments/assets/2bc5969a-152e-47a7-8472-15818a32888e" />
