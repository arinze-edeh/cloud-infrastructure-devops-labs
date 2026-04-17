# Terraform AWS IAM Group Provisioning via LocalStack

> **Domain:** Identity and Access Management | Infrastructure as Code
> **Tool:** Terraform v1.11.0
> **Provider:** hashicorp/aws v5.91.0
> **Environment:** LocalStack (AWS Emulation)
> **Working Directory:** `/home/bob/terraform`

---

## Table of Contents

* [Overview](#overview)
* [Business Context](#business-context)
* [Architecture and Design](#architecture-and-design)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify the Terraform Environment](#step-1-verify-the-terraform-environment)
  * [Step 2: Review the Provider Configuration](#step-2-review-the-provider-configuration)
  * [Step 3: Author the IAM Group Resource Configuration](#step-3-author-the-iam-group-resource-configuration)
  * [Step 4: Initialize the Terraform Working Directory](#step-4-initialize-the-terraform-working-directory)
  * [Step 5: Validate the Configuration](#step-5-validate-the-configuration)
  * [Step 6: Generate and Review the Execution Plan](#step-6-generate-and-review-the-execution-plan)
  * [Step 7: Apply the Configuration](#step-7-apply-the-configuration)
  * [Step 8: Verify the Provisioned Resource](#step-8-verify-the-provisioned-resource)
* [Resource Verification](#resource-verification)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [Troubleshooting Reference](#troubleshooting-reference)

---

## Overview

This project provisions an AWS IAM Group named `iamgroup_james` using Terraform against a LocalStack-emulated AWS environment. It demonstrates a disciplined Infrastructure as Code workflow covering configuration authoring, provider initialization, plan review, and deterministic apply with full state verification.

The implementation follows the principle of minimal, purpose-scoped Terraform configurations to maximize clarity, reproducibility, and auditability for teams managing identity resources at scale.

---

## Business Context

The James DevOps team is executing a phased AWS cloud migration. To maintain granular access control, risk mitigation, and resource optimization across migration stages, infrastructure resources are provisioned in discrete, independently verifiable units.

This task addresses the identity layer requirement: establishing the IAM group `iamgroup_james` as a foundation for attaching policies and managing team-level permissions programmatically through Terraform rather than through manual console operations.

---

## Architecture and Design

```
LocalStack (http://aws:4566)
        |
        | Terraform AWS Provider (v5.91.0)
        |
        v
  AWS IAM Group
  Name: iamgroup_james
  Path: /
```

All AWS service endpoints are routed to the LocalStack container at `http://aws:4566`. This mirrors a real AWS IAM provisioning flow while enabling fully isolated, credential-free execution in the lab environment. The `skip_credentials_validation` and `skip_requesting_account_id` provider flags are specific to LocalStack and must not be carried into production AWS configurations.

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform | v1.11.0 (installed) |
| AWS Provider | hashicorp/aws v5.91.0 |
| LocalStack | Running at `http://aws:4566` |
| Working Directory | `/home/bob/terraform` |
| Shell Access | Terminal with write permissions to working directory |

---

## Repository Structure

```
/home/bob/terraform/
├── provider.tf          # AWS provider and backend configuration
├── main.tf              # IAM group resource definition (authored in this task)
├── README.MD            # Pre-existing task reference document
├── .terraform/          # Provider plugins (generated after init)
└── .terraform.lock.hcl  # Dependency lock file (generated after init)
```

> **Note:** Only `main.tf` was created during this implementation. All other files were pre-existing in the working directory. The task requirement explicitly mandates that no additional `.tf` files be created; all resource definitions must reside in `main.tf`.

---

## Implementation Guide

### Step 1: Verify the Terraform Environment

Confirm the Terraform binary version and inspect the existing working directory before making any changes.

```bash
ls -la
```

**Output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 17 05:02 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

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

> **Note:** The environment runs Terraform v1.11.0. The version warning is informational and does not affect the execution of this configuration. The provider version pinned in `provider.tf` (v5.91.0) is fully compatible with this Terraform runtime.

Screenshot: `terraform version output confirming v1.11.0 on linux_amd64`

<img width="1045" height="591" alt="image" src="https://github.com/user-attachments/assets/8efb6004-15ab-4970-ae68-2143576de1ce" />

---

### Step 2: Review the Provider Configuration

Read the existing `provider.tf` to understand the backend setup and endpoint routing before authoring any new resources.

```bash
cat provider.tf
```

**Output:**

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

**Key observations from provider review:**

* The `iam` endpoint is correctly routed to LocalStack at `http://aws:4566`, confirming that `aws_iam_group` resources will be provisioned against the emulated backend.
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are LocalStack-specific flags that bypass real AWS credential checks.
* The provider version is pinned to `5.91.0`, ensuring reproducible provider behavior across all team members and CI runs.

Screenshot: `provider.tf contents displayed in terminal`

---

### Step 3: Author the IAM Group Resource Configuration

Open `main.tf` for editing and define the IAM group resource. This is the only file created during this implementation.

```bash
vi main.tf
```

Enter the following resource block:

```hcl
resource "aws_iam_group" "james_group" {
  name = "iamgroup_james"
}
```

Verify the file contents after saving:

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_iam_group" "james_group" {
  name = "iamgroup_james"
}
```

**Configuration breakdown:**

| Field | Value | Purpose |
|---|---|---|
| Resource type | `aws_iam_group` | Terraform resource for AWS IAM Group |
| Resource label | `james_group` | Local Terraform identifier for state references |
| `name` | `iamgroup_james` | The IAM group name as it will appear in AWS |

Screenshot: `main.tf opened in vi with the aws_iam_group resource block`

---

### Step 4: Initialize the Terraform Working Directory

Run `terraform init` to download the required provider plugin and prepare the working directory. This command must be run before any plan or apply operation.

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

The `.terraform.lock.hcl` file is generated at this stage and records the exact provider version resolved. This file must be committed to version control to guarantee consistent provider resolution across team environments.

Screenshot: `terraform init completing successfully with provider v5.91.0 installed`

---

### Step 5: Validate the Configuration

Run `terraform validate` to perform a static syntax and schema check against the configuration files without making any API calls.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

Validation confirms that the `aws_iam_group` resource block is syntactically correct and all required arguments are present according to the provider schema.

Screenshot: `terraform validate returning Success with no errors`

---

### Step 6: Generate and Review the Execution Plan

Run `terraform plan` to produce a detailed preview of the changes Terraform will make. Review the plan carefully before proceeding to apply.

```bash
terraform plan
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_group.james_group will be created
  + resource "aws_iam_group" "james_group" {
      + arn       = (known after apply)
      + id        = (known after apply)
      + name      = "iamgroup_james"
      + path      = "/"
      + unique_id = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Plan analysis:**

* **1 resource to add:** The IAM group `iamgroup_james` will be created.
* **0 changes, 0 destructions:** No existing state is affected.
* **Computed attributes:** `arn`, `id`, and `unique_id` are resolved at apply time by the AWS API.
* **Default path:** The group is created at the IAM root path `/` as expected when no explicit `path` argument is provided.

Screenshot: `terraform plan output showing + create for aws_iam_group.james_group`

---

### Step 7: Apply the Configuration

Execute `terraform apply` to provision the IAM group. Confirm the operation when prompted.

```bash
terraform apply
```

At the interactive prompt, type `yes` to approve:

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Output:**

```
aws_iam_group.james_group: Creating...
aws_iam_group.james_group: Creation complete after 0s [id=iamgroup_james]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The IAM group was created in 0 seconds, consistent with LocalStack's in-memory emulation latency. In a real AWS environment, IAM resource creation typically completes within 1 to 3 seconds.

Screenshot: `terraform apply completing with Apply complete! Resources: 1 added, 0 changed, 0 destroyed`

---

### Step 8: Verify the Provisioned Resource

Use `terraform state show` to inspect the full resource attributes recorded in the Terraform state file post-apply. This is the authoritative verification step confirming the resource was created with the correct attributes.

```bash
terraform state show aws_iam_group.james_group
```

**Output:**

```hcl
# aws_iam_group.james_group:
resource "aws_iam_group" "james_group" {
    arn       = "arn:aws:iam::000000000000:group/iamgroup_james"
    id        = "iamgroup_james"
    name      = "iamgroup_james"
    unique_id = "ggg9l5em15d6hlvxrilr"
    path      = "/"
}
```

**State verification table:**

| Attribute | Value | Notes |
|---|---|---|
| `arn` | `arn:aws:iam::000000000000:group/iamgroup_james` | LocalStack uses account ID `000000000000` |
| `id` | `iamgroup_james` | Matches the `name` attribute as expected for IAM groups |
| `name` | `iamgroup_james` | Confirmed correct per task requirement |
| `path` | `/` | Default IAM root path |
| `unique_id` | `ggg9l5em15d6hlvxrilr` | Unique identifier generated by LocalStack |

Screenshot: `terraform state show output displaying all attributes of aws_iam_group.james_group`

---

## Resource Verification

The full provisioning lifecycle completed without errors:

| Stage | Command | Result |
|---|---|---|
| Environment check | `terraform version` | v1.11.0 confirmed |
| Directory inspection | `ls -la` | Working directory confirmed clean |
| Provider review | `cat provider.tf` | IAM endpoint confirmed at LocalStack |
| Resource authoring | `vi main.tf` | `aws_iam_group` resource defined |
| Initialization | `terraform init` | Provider v5.91.0 installed and locked |
| Validation | `terraform validate` | Configuration valid |
| Plan | `terraform plan` | 1 resource to add, 0 changes, 0 destructions |
| Apply | `terraform apply` | 1 resource added successfully |
| State verification | `terraform state show` | All attributes confirmed in state |

---

## Best Practices Applied

* **Provider version pinning:** The `version = "5.91.0"` constraint in `required_providers` prevents unintended provider upgrades during future `terraform init` runs, ensuring reproducible behavior across team environments and CI/CD pipelines.

* **Lock file committed to version control:** The `.terraform.lock.hcl` file generated by `terraform init` records the exact provider hash and version. Committing this file guarantees that all consumers of the repository resolve the identical provider binary.

* **Validate before plan:** Running `terraform validate` before `terraform plan` catches syntax and schema errors without making any API calls, reducing unnecessary network round-trips in CI pipelines.

* **Explicit plan review before apply:** The execution plan was reviewed in full before `terraform apply` was invoked. In production workflows, the `-out=tfplan` flag should be used to save the plan file and pass it to `apply` directly, preventing plan drift between the review and execution stages.

* **State verification post-apply:** `terraform state show` was used to confirm the provisioned resource attributes from the Terraform state file, providing an auditable record independent of console inspection.

* **Single-file resource organization:** All resource definitions are contained within `main.tf` as required, keeping the configuration surface area minimal and reducing cognitive overhead for reviewers.

* **Separation of provider and resource configuration:** The `provider.tf` and `main.tf` files maintain a clean separation between infrastructure plumbing (provider setup) and resource declarations, aligning with team maintainability standards.

---

## Lessons Learned

* **LocalStack account ID is always `000000000000`:** The ARN generated for the IAM group reflects `arn:aws:iam::000000000000:group/iamgroup_james`. Teams validating ARN formats in tests or policy documents should account for this LocalStack convention and not mistake it for a misconfiguration.

* **Computed attributes are unavailable until apply:** The `arn`, `id`, and `unique_id` fields displayed as `(known after apply)` in the plan output. Referencing these values in other resources within the same configuration requires either a `depends_on` declaration or direct interpolation using the resource reference (for example, `aws_iam_group.james_group.arn`), which Terraform resolves correctly post-apply.

* **`terraform validate` does not make API calls:** This is a local-only check. It does not validate that provider endpoints are reachable or that the IAM group name is globally unique. Connectivity and naming conflicts surface only at `terraform plan` or `terraform apply` time.

* **LocalStack applies near-instant provisioning:** The `Creation complete after 0s` message reflects LocalStack's in-memory execution model. In real AWS environments, IAM changes propagate globally and may have eventual consistency delays of a few seconds. Downstream automation that immediately queries IAM after provisioning should account for this.

* **`vi` is the available editor in restricted environments:** In hardened lab environments where graphical editors and IDE terminals are unavailable, `vi` proficiency is required. Alternatively, `cat > main.tf << 'EOF'` heredoc syntax can be used to write files non-interactively in scripts.

---

## Troubleshooting Reference

| Symptom | Root Cause | Resolution |
|---|---|---|
| `Error: No valid credential sources found` | Missing LocalStack flags | Confirm `skip_credentials_validation = true` and `skip_requesting_account_id = true` in `provider.tf` |
| `Error: Failed to install provider` | Network unreachable to Terraform registry | Verify internet connectivity or use a Terraform plugin cache |
| `Error: Invalid resource type` | Provider not initialized | Run `terraform init` before `terraform plan` or `terraform apply` |
| `Error: Group iamgroup_james already exists` | Resource exists in LocalStack state from a prior run | Run `terraform state list` to inspect existing state, or `terraform import` to bring the resource under management |
| Plan shows no changes after editing `main.tf` | Cached state reflects already-provisioned resource | Inspect state with `terraform state show`; if resource exists, no apply is needed |











<img width="1026" height="527" alt="image" src="https://github.com/user-attachments/assets/0c9694f2-763e-4afc-b8b9-2a323294bdd0" />

<img width="1046" height="768" alt="image" src="https://github.com/user-attachments/assets/4c105d1b-cbe2-4c4a-89ad-ff21f108f53d" />
<img width="1038" height="766" alt="image" src="https://github.com/user-attachments/assets/41a04851-3316-4b09-84ec-7bd7323e6522" />
<img width="1050" height="771" alt="image" src="https://github.com/user-attachments/assets/16a60360-ec6c-453a-acea-796ee02b86b0" />
<img width="1050" height="598" alt="image" src="https://github.com/user-attachments/assets/ade8ce2a-7098-4887-975a-4827d6f312a4" />
<img width="1047" height="529" alt="image" src="https://github.com/user-attachments/assets/3339c337-da67-4021-a166-430dd6c38904" />
<img width="1047" height="606" alt="image" src="https://github.com/user-attachments/assets/a01cb9f3-e0be-4230-880e-e19613b6961d" />
<img width="1050" height="527" alt="image" src="https://github.com/user-attachments/assets/31b18a74-5753-43bc-9f37-8b9264172e7f" />
<img width="1053" height="695" alt="image" src="https://github.com/user-attachments/assets/bea5ef63-8278-49c6-96a2-ce3cac4e08cd" />
<img width="1046" height="720" alt="image" src="https://github.com/user-attachments/assets/5782798d-2324-4746-a211-dd328917bd47" />
