# Terraform IAM User Provisioning on AWS (LocalStack)

> **Platform:** KodeKloud / Nautilus DevOps Lab
> **Domain:** Identity and Access Management (IAM)
> **Toolchain:** Terraform, AWS CLI, LocalStack
> **Difficulty:** Foundational

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Inspect the Working Directory](#step-1-inspect-the-working-directory)
  - [Step 2: Review the Existing Provider Configuration](#step-2-review-the-existing-provider-configuration)
  - [Step 3: Author the Resource Configuration](#step-3-author-the-resource-configuration)
  - [Step 4: Verify Configuration Files](#step-4-verify-configuration-files)
  - [Step 5: Initialize the Terraform Working Directory](#step-5-initialize-the-terraform-working-directory)
  - [Step 6: Validate the Configuration](#step-6-validate-the-configuration)
  - [Step 7: Generate and Review the Execution Plan](#step-7-generate-and-review-the-execution-plan)
  - [Step 8: Apply the Configuration](#step-8-apply-the-configuration)
  - [Step 9: Verify Terraform State](#step-9-verify-terraform-state)
  - [Step 10: Confirm Resource Creation via AWS CLI](#step-10-confirm-resource-creation-via-aws-cli)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors and Resolutions](#errors-and-resolutions)
- [Screenshots](#screenshots)
- [References](#references)

---

## Overview

This lab demonstrates the provisioning of an AWS IAM user using Terraform within a LocalStack-simulated AWS environment. IAM is one of the most foundational services in any AWS deployment, providing the access control layer that governs every interaction with cloud resources. This implementation follows infrastructure-as-code (IaC) principles, ensuring repeatable, auditable, and version-controlled identity provisioning.

---

## Problem Statement

The Nautilus DevOps team is in the process of configuring AWS Identity and Access Management resources on their cloud environment. IAM enables the creation and lifecycle management of user accounts, groups, roles, and policies, forming the security backbone of any AWS workload.

**Task requirement:** Create an IAM user named `iamuser_jim` using Terraform. The Terraform working directory is `/home/bob/terraform`. The resource definition must be placed exclusively in `main.tf`. No additional `.tf` files may be created for this task.

---

## Architecture and Design Intent

```
LocalStack (http://aws:4566)
        |
        | Terraform AWS Provider (v5.91.0)
        |
        v
  AWS IAM Service (simulated)
        |
        v
  IAM User: iamuser_jim
        Path: /
        ARN:  arn:aws:iam::000000000000:user/iamuser_jim
```

LocalStack simulates the AWS API surface on `http://aws:4566`, enabling fully offline Terraform workflows without incurring cloud costs or requiring real AWS credentials. All Terraform resource operations target this local endpoint.

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform | Installed and available in `$PATH` |
| AWS Provider | `hashicorp/aws` v5.91.0 (pinned in `provider.tf`) |
| AWS CLI | Configured to target LocalStack endpoint |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |

---

## Repository Structure

```
/home/bob/terraform/
    main.tf           # IAM user resource definition (authored in this task)
    provider.tf       # AWS provider and LocalStack endpoint configuration
    README.MD         # Original lab readme
    .terraform/       # Provider plugins (generated after terraform init)
    .terraform.lock.hcl  # Dependency lock file (generated after terraform init)
```

---

## Implementation Guide

### Step 1: Inspect the Working Directory

Before writing any configuration, inspect the existing workspace to understand the current state of the directory.

```bash
ls -la
```

**Output observed:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 16 00:13 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

The directory contains only `provider.tf` and the lab `README.MD`. No `main.tf` exists yet. The provider configuration is already in place and does not need to be modified.

> Screenshot: 

<img width="1038" height="578" alt="image" src="https://github.com/user-attachments/assets/a403401d-cc6d-490f-9ffd-e1bd036ec85f" />

---

### Step 2: Review the Existing Provider Configuration

Read the pre-existing `provider.tf` to understand the provider version constraint, the target region, and the LocalStack endpoint mappings before authoring any resource configuration.

```bash
cat ~/terraform/provider.tf
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

**Key observations:**

* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack compatibility since no real AWS credentials are present.
* The `iam` endpoint is mapped to `http://aws:4566`, confirming that all IAM API calls will be intercepted by LocalStack.
* The provider version is pinned to `5.91.0` to ensure reproducibility.

> Screenshot: 

<img width="1074" height="801" alt="image" src="https://github.com/user-attachments/assets/8c8e01ed-048a-4362-bb2b-52a23a3c4b28" />

---

### Step 3: Author the Resource Configuration

Create `main.tf` with the IAM user resource definition. Per the task constraint, this is the only `.tf` file that may be created.

```bash
cat > ~/terraform/main.tf << 'EOF'
resource "aws_iam_user" "iamuser_jim" {
  name = "iamuser_jim"
}
EOF
```

Verify the file was written correctly:

```bash
cat ~/terraform/main.tf
```

**Output observed:**

```hcl
resource "aws_iam_user" "iamuser_jim" {
  name = "iamuser_jim"
}
```

**Design notes:**

* The Terraform resource label `iamuser_jim` mirrors the IAM username for clarity and traceability.
* No additional arguments (`path`, `tags`, `permissions_boundary`) are specified because the task requires only the user creation with the specified name. The `path` defaults to `/` as confirmed in the plan output.

> Screenshot: 

<img width="1045" height="677" alt="image" src="https://github.com/user-attachments/assets/30c56ad9-9701-4353-89d2-1efe093499ed" />

---

### Step 4: Verify Configuration Files

Confirm that both required `.tf` files are present and no extraneous files were created.

```bash
ls -1 ~/terraform/*.tf
```

**Output observed:**

```
/home/bob/terraform/main.tf
/home/bob/terraform/provider.tf
```

Only the two expected files are present. This satisfies the task constraint of not creating additional `.tf` files.

> Screenshot: 

<img width="1046" height="304" alt="image" src="https://github.com/user-attachments/assets/6b2928ae-767f-4dd0-b09b-bd722f0a1bd5" />

---

### Step 5: Initialize the Terraform Working Directory

Run `terraform init` to download the pinned AWS provider plugin and create the `.terraform` directory and `.terraform.lock.hcl` lock file.

```bash
cd ~/terraform && terraform init
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

`terraform init` resolved and installed the `hashicorp/aws` provider at the exact pinned version `5.91.0`. The lock file was created to enforce deterministic provider resolution in subsequent runs.

> Screenshot: 

<img width="1045" height="677" alt="image" src="https://github.com/user-attachments/assets/7edfe72d-dae3-4022-8c79-ef217c6d1592" />

---

### Step 6: Validate the Configuration

Run `terraform validate` to perform a static syntax and semantic check of all `.tf` files before attempting a plan or apply.

```bash
terraform validate
```

**Output observed:**

```
Success! The configuration is valid.
```

No syntax errors, undefined references, or structural issues were detected. This step is a pre-flight check that catches authoring mistakes before any API calls are made.

> Screenshot: 

<img width="1043" height="642" alt="image" src="https://github.com/user-attachments/assets/09765f08-a903-4511-9808-a23a1d14ce65" />

---

### Step 7: Generate and Review the Execution Plan

Run `terraform plan` to preview the exact changes Terraform will make. This is a non-destructive, read-only operation that contacts the state backend and the provider.

```bash
terraform plan
```

**Output observed:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_user.iamuser_jim will be created
  + resource "aws_iam_user" "iamuser_jim" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "iamuser_jim"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Plan analysis:**

* Terraform identified exactly one resource to create: `aws_iam_user.iamuser_jim`.
* `force_destroy = false` is the safe default, meaning Terraform will not forcibly delete the user if it has attached policies or group memberships at destroy time.
* `path = "/"` is the default IAM path, placing the user in the root namespace.
* Attributes marked `(known after apply)` are server-assigned values such as `arn`, `id`, and `unique_id`, which will be populated post-creation.

> Screenshot: `07-terraform-plan.png`

---

### Step 8: Apply the Configuration

Apply the plan with auto-approval to suppress the interactive confirmation prompt. In lab environments this is acceptable; in production workflows, the `-auto-approve` flag must be used with explicit team authorization.

```bash
terraform apply -auto-approve
```

**Output observed:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_user.iamuser_jim will be created
  + resource "aws_iam_user" "iamuser_jim" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "iamuser_jim"
      + path          = "/"
      + tags_all      = (known after apply)
      + unique_id     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_user.iamuser_jim: Creating...
aws_iam_user.iamuser_jim: Creation complete after 0s [id=iamuser_jim]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The IAM user `iamuser_jim` was created successfully in under one second. The resource ID is `iamuser_jim`, consistent with the `name` attribute defined in `main.tf`.

> Screenshot: `08-terraform-apply.png`

---

### Step 9: Verify Terraform State

Query the Terraform state to confirm the resource is tracked correctly in the local state file.

```bash
terraform state list
```

**Output observed:**

```
aws_iam_user.iamuser_jim
```

The resource address `aws_iam_user.iamuser_jim` is recorded in the state file. This confirms Terraform has full lifecycle ownership of the resource and will manage future changes or destruction operations against it.

> Screenshot: `09-terraform-state-list.png`

---

### Step 10: Confirm Resource Creation via AWS CLI

Independently verify the IAM user exists in LocalStack by querying the IAM API directly using the AWS CLI with the LocalStack endpoint override.

```bash
aws --endpoint-url=http://aws:4566 iam list-users
```

**Output observed:**

```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "iamuser_jim",
            "UserId": "59n1d0hjtrfyn54xibi5",
            "Arn": "arn:aws:iam::000000000000:user/iamuser_jim",
            "CreateDate": "2026-04-16T00:29:26.306313Z"
        }
    ]
}
```

**Verification analysis:**

* `UserName: iamuser_jim` matches the required resource name exactly.
* `Path: /` reflects the default IAM path as expected.
* `Arn: arn:aws:iam::000000000000:user/iamuser_jim` uses LocalStack's placeholder account ID `000000000000`.
* `CreateDate` confirms the user was created during this lab session.

This dual verification (Terraform state + AWS CLI) constitutes a complete end-to-end validation of the provisioning workflow.

> Screenshot: `10-aws-cli-iam-list-users.png`

---

## Best Practices Applied

* **Provider version pinning:** The AWS provider is locked to `5.91.0` in `provider.tf`, preventing unexpected behavior from automatic upgrades across team environments.
* **Lock file inclusion:** `.terraform.lock.hcl` is generated by `terraform init` and should be committed to version control to guarantee provider version consistency for all contributors.
* **Validate before plan:** Running `terraform validate` prior to `terraform plan` catches configuration authoring errors early without consuming API quota or triggering network calls.
* **Plan before apply:** Reviewing the execution plan allows operators to confirm the exact diff before any infrastructure changes are committed. This is especially critical in production where the blast radius of unreviewed changes can be significant.
* **Separate provider and resource files:** Keeping `provider.tf` and `main.tf` distinct follows the standard Terraform file layout convention, improving maintainability as the configuration grows.
* **Out-of-band verification:** Confirming resource creation via the AWS CLI independently of Terraform validates both the Terraform apply result and the underlying API, ensuring the resource physically exists and is not simply recorded in state due to a provider bug.
* **Minimal resource definition:** Only the required `name` argument is specified. Omitting optional arguments that carry acceptable defaults keeps the configuration clean and reduces configuration drift risk.

---

## Lessons Learned

* **LocalStack endpoint configuration is comprehensive by design.** The `endpoints` block in `provider.tf` maps every AWS service to `http://aws:4566`. In real AWS environments, this block is absent because the provider resolves endpoints automatically. Understanding this distinction is essential when transitioning lab configurations to production.

* **`force_destroy = false` is the safe production default for IAM users.** When an IAM user has attached policies, group memberships, or access keys, Terraform will fail to destroy it unless `force_destroy = true` is explicitly set. This default prevents accidental destructive operations and should only be overridden with deliberate intent.

* **The `terraform state list` command is the quickest way to confirm resource tracking.** After any apply operation, checking state ensures Terraform has registered the resource for future lifecycle management. A resource that exists in the cloud but not in state is invisible to Terraform and can lead to duplicate resource creation or drift.

* **The `-auto-approve` flag bypasses the human review gate.** In team and production workflows, `terraform apply` should always be run interactively or with a mandatory plan review step enforced by CI/CD pipelines (e.g., Atlantis, Terraform Cloud). The `-auto-approve` shortcut is appropriate only in isolated lab environments.

* **`skip_credentials_validation` and `skip_requesting_account_id` are LocalStack-specific.** These provider arguments disable AWS credential checks that would fail in an environment with no real AWS account. They must not be carried into production provider configurations.

---

## Errors and Resolutions

No errors were encountered during this implementation. The following pre-conditions ensured a clean execution:

* The `provider.tf` was pre-configured correctly with all required LocalStack endpoint mappings.
* The AWS provider version `5.91.0` was available in the Terraform registry and installed successfully on first `init`.
* The resource definition in `main.tf` contained no syntax errors, confirmed by `terraform validate`.

**Potential failure modes to be aware of:**

| Scenario | Symptom | Resolution |
|---|---|---|
| LocalStack not running | `terraform apply` returns connection refused on `http://aws:4566` | Start LocalStack before running Terraform commands |
| Provider version unavailable | `terraform init` fails with version constraint error | Verify registry connectivity and correct version string in `required_providers` |
| Duplicate resource name | `aws_iam_user` with same `name` already exists in state | Run `terraform state list` to check, then import or destroy the conflicting resource |
| Wrong working directory | `terraform` commands run outside `/home/bob/terraform` | Always `cd` into the Terraform working directory before running any commands |

---

## Screenshots

| Step | Screenshot File | Description |
|---|---|---|
| 1 | `01-initial-directory-listing.png` | Output of `ls -la` showing only `provider.tf` and `README.MD` present |
| 2 | `02-provider-tf-contents.png` | Full contents of `provider.tf` including LocalStack endpoint mappings |
| 3 | `03-main-tf-authored.png` | Heredoc command and `cat` verification of `main.tf` contents |
| 4 | `04-tf-files-verified.png` | `ls -1 ~/terraform/*.tf` confirming only two `.tf` files exist |
| 5 | `05-terraform-init.png` | `terraform init` output confirming provider installation and lock file creation |
| 6 | `06-terraform-validate.png` | `terraform validate` output showing `Success! The configuration is valid.` |
| 7 | `07-terraform-plan.png` | `terraform plan` output showing the single `+ create` action for `aws_iam_user.iamuser_jim` |
| 8 | `08-terraform-apply.png` | `terraform apply -auto-approve` output confirming `Apply complete! Resources: 1 added` |
| 9 | `09-terraform-state-list.png` | `terraform state list` confirming `aws_iam_user.iamuser_jim` is tracked |
| 10 | `10-aws-cli-iam-list-users.png` | AWS CLI `iam list-users` JSON response confirming `iamuser_jim` exists in LocalStack |

---

## References

* [Terraform AWS Provider Documentation: aws_iam_user](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user)
* [Terraform CLI: init](https://developer.hashicorp.com/terraform/cli/commands/init)
* [Terraform CLI: validate](https://developer.hashicorp.com/terraform/cli/commands/validate)
* [Terraform CLI: plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
* [Terraform CLI: apply](https://developer.hashicorp.com/terraform/cli/commands/apply)
* [Terraform CLI: state list](https://developer.hashicorp.com/terraform/cli/commands/state/list)
* [LocalStack Documentation](https://docs.localstack.cloud/overview/)
* [AWS IAM User Management Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

---

*Authored by Arinze Edeh | Cloud and DevOps Engineer | GitHub: [arinze-edeh](https://github.com/arinze-edeh)*










<img width="1051" height="574" alt="image" src="https://github.com/user-attachments/assets/6930ddb9-2a12-48f1-95ec-b2f85a2597e0" />
<img width="1037" height="533" alt="image" src="https://github.com/user-attachments/assets/8154162c-7a9c-48af-9fc4-fd0dc5d46232" />
<img width="1046" height="591" alt="image" src="https://github.com/user-attachments/assets/100a741f-8c49-4ca6-92e6-466784e5b867" />
<img width="1048" height="352" alt="image" src="https://github.com/user-attachments/assets/2349df51-91dc-4a92-8802-47f1ce6c4255" />



