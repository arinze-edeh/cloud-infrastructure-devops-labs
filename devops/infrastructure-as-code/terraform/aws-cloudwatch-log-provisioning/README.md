# Terraform AWS CloudWatch Log Group and Log Stream Provisioning

> Automated provisioning of CloudWatch logging infrastructure using Terraform against a LocalStack-emulated AWS environment, implementing enterprise-grade IaC patterns for observable, audit-ready cloud workloads.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Phase 1: Environment Verification](#phase-1-environment-verification)
  - [Phase 2: Terraform Configuration Authoring](#phase-2-terraform-configuration-authoring)
  - [Phase 3: Initialization and Validation](#phase-3-initialization-and-validation)
  - [Phase 4: Plan and Apply](#phase-4-plan-and-apply)
  - [Phase 5: State Verification](#phase-5-state-verification)
- [Resource Inventory](#resource-inventory)
- [Best Practices Applied](#best-practices-applied)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [References](#references)

---

## Overview

This implementation provisions a CloudWatch Log Group and a CloudWatch Log Stream on a LocalStack-emulated AWS endpoint using Terraform as the infrastructure-as-code engine. The solution is authored in a single `main.tf` file within an existing Terraform working directory that already contains a pre-configured `provider.tf` targeting LocalStack.

**Business context:** The Nautilus DevOps team requires a structured logging infrastructure for their application. Centralizing logs under a named log group with a dedicated log stream enables structured observability, log-based alerting, and audit traceability across distributed workloads.

**Key outcomes:**
* CloudWatch Log Group `xfusion-log-group` provisioned and registered in Terraform state
* CloudWatch Log Stream `xfusion-log-stream` provisioned and linked to the log group
* Full Terraform lifecycle executed: `init` > `validate` > `plan` > `apply`
* Infrastructure state confirmed via `terraform state list`

---

## Architecture

```
LocalStack (http://aws:4566)
    |
    +-- CloudWatch
            |
            +-- Log Group: xfusion-log-group
                    |
                    +-- Log Stream: xfusion-log-stream
```

The Log Stream is declared with an explicit dependency on the Log Group via the `log_group_name` attribute referencing the Terraform resource attribute `aws_cloudwatch_log_group.xfusion_log_group.name`. This enforces the correct provisioning order without requiring an explicit `depends_on` block.

---

## Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform | >= 1.x |
| AWS Provider | hashicorp/aws 5.91.0 |
| LocalStack | Running and accessible at `http://aws:4566` |
| User account | `bob` with write access to `/home/bob/terraform` |
| Working directory | `/home/bob/terraform` |

The `provider.tf` file must already exist in the working directory and must configure the `aws` provider with LocalStack endpoint overrides for all required services, including `cloudwatch`.

---

## Repository Structure

```
/home/bob/terraform/
|-- provider.tf          # Pre-existing: AWS provider configuration targeting LocalStack
|-- main.tf              # Authored in this implementation: CloudWatch resource definitions
|-- README.MD            # Pre-existing: task description
|-- .terraform/          # Auto-generated: provider plugin cache (post-init)
|-- .terraform.lock.hcl  # Auto-generated: provider dependency lock file (post-init)
|-- terraform.tfstate    # Auto-generated: infrastructure state (post-apply)
```

---

## Implementation Guide

### Phase 1: Environment Verification

Before authoring any configuration, the working directory contents and active user context were verified to confirm the environment was in a known-good state.

**List working directory contents:**

```bash
ls -la
```

**Expected output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 23 21:37 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

**Verify active user:**

```bash
whoami
```

**Expected output:**

```
bob
```

**Inspect the pre-existing provider configuration:**

```bash
cat provider.tf
```

**Expected output:**

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

> Screenshot: Terminal output showing `ls -la`, `whoami`, and `cat provider.tf` results confirming the environment state before authoring begins

<img width="1074" height="793" alt="image" src="https://github.com/user-attachments/assets/2ca1d647-8f58-4914-aeea-92c04d3f4c79" />

---

### Phase 2: Terraform Configuration Authoring

A new `main.tf` file was created in the working directory using a heredoc redirect. No additional `.tf` files were created; all resource definitions are consolidated in this single file as required.

**Create `main.tf` with the CloudWatch resource definitions:**

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_cloudwatch_log_group" "xfusion_log_group" {
  name = "xfusion-log-group"
}

resource "aws_cloudwatch_log_stream" "xfusion_log_stream" {
  name           = "xfusion-log-stream"
  log_group_name = aws_cloudwatch_log_group.xfusion_log_group.name
}
EOF
```

**Verify the file was written correctly:**

```bash
cat main.tf
```

**Expected output:**

```hcl
resource "aws_cloudwatch_log_group" "xfusion_log_group" {
  name = "xfusion-log-group"
}

resource "aws_cloudwatch_log_stream" "xfusion_log_stream" {
  name           = "xfusion-log-stream"
  log_group_name = aws_cloudwatch_log_group.xfusion_log_group.name
}
```

> Screenshot: Terminal output showing the `cat main.tf` result confirming accurate heredoc write with both resource blocks

<img width="1043" height="737" alt="image" src="https://github.com/user-attachments/assets/c8b59294-a82f-4185-bf47-2e8e0c017872" />

**Configuration notes:**

* The Log Group resource identifier is `aws_cloudwatch_log_group.xfusion_log_group` with `name = "xfusion-log-group"`.
* The Log Stream resource identifier is `aws_cloudwatch_log_stream.xfusion_log_stream` with `name = "xfusion-log-stream"`.
* The `log_group_name` attribute on the Log Stream references the Log Group by its Terraform attribute `aws_cloudwatch_log_group.xfusion_log_group.name`, establishing an implicit resource dependency. This guarantees the Log Group is created before Terraform attempts to create the Log Stream.

---

### Phase 3: Initialization and Validation

**Initialize the Terraform working directory:**

```bash
terraform init
```

**Expected output:**

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

> Screenshot: Terminal output showing successful `terraform init` with provider installation confirmation and lock file creation

<img width="1045" height="582" alt="image" src="https://github.com/user-attachments/assets/1f2ac77d-fc0d-4faa-b263-1c1245538c89" />

**Validate the configuration syntax and schema:**

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

> Screenshot: Terminal output confirming `terraform validate` passes with no errors or warnings

<img width="1044" height="473" alt="image" src="https://github.com/user-attachments/assets/3efe1cc2-6c89-43a3-b2d7-d9f0ba18c9b0" />

---

### Phase 4: Plan and Apply

**Generate and review the execution plan:**

```bash
terraform plan
```

**Expected output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_cloudwatch_log_group.xfusion_log_group will be created
  + resource "aws_cloudwatch_log_group" "xfusion_log_group" {
      + arn               = (known after apply)
      + id                = (known after apply)
      + log_group_class   = (known after apply)
      + name              = "xfusion-log-group"
      + name_prefix       = (known after apply)
      + retention_in_days = 0
      + skip_destroy      = false
      + tags_all          = (known after apply)
    }

  # aws_cloudwatch_log_stream.xfusion_log_stream will be created
  + resource "aws_cloudwatch_log_stream" "xfusion_log_stream" {
      + arn            = (known after apply)
      + id             = (known after apply)
      + log_group_name = "xfusion-log-group"
      + name           = "xfusion-log-stream"
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

> Screenshot: Terminal output showing the full `terraform plan` execution plan with 2 resources to add and 0 to change or destroy

<img width="1046" height="729" alt="image" src="https://github.com/user-attachments/assets/d531fd46-43fc-4e0e-839d-fd1c1640a192" />

**Apply the configuration:**

```bash
terraform apply -auto-approve
```

**Expected output:**

```
Terraform used the selected providers to generate the following execution plan.

Plan: 2 to add, 0 to change, 0 to destroy.
aws_cloudwatch_log_group.xfusion_log_group: Creating...
aws_cloudwatch_log_group.xfusion_log_group: Creation complete after 1s [id=xfusion-log-group]
aws_cloudwatch_log_stream.xfusion_log_stream: Creating...
aws_cloudwatch_log_stream.xfusion_log_stream: Creation complete after 0s [id=xfusion-log-stream]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

> Screenshot: Terminal output showing `terraform apply -auto-approve` completing successfully with both resources created in dependency order

<img width="1046" height="736" alt="image" src="https://github.com/user-attachments/assets/3abae4d4-f927-4a0a-ac6f-6905e751ce1a" />

**Observed provisioning sequence:**

1. `aws_cloudwatch_log_group.xfusion_log_group` created first (1 second)
2. `aws_cloudwatch_log_stream.xfusion_log_stream` created second (0 seconds), after the Log Group was fully registered

This ordering confirms Terraform correctly resolved the implicit dependency declared through the attribute reference.

---

### Phase 5: State Verification

**Confirm both resources are tracked in Terraform state:**

```bash
terraform state list
```

**Expected output:**

```
aws_cloudwatch_log_group.xfusion_log_group
aws_cloudwatch_log_stream.xfusion_log_stream
```

> Screenshot: Terminal output showing `terraform state list` with both provisioned resources confirmed in state

<img width="1044" height="381" alt="image" src="https://github.com/user-attachments/assets/d652faf2-625f-4bf0-be6d-9246b0cddd5f" />

Both resources are present in the state file, confirming the apply was fully successful and the infrastructure is now managed by Terraform.

---

## Resource Inventory

| Resource Type | Terraform Resource ID | AWS Resource Name |
|---|---|---|
| `aws_cloudwatch_log_group` | `xfusion_log_group` | `xfusion-log-group` |
| `aws_cloudwatch_log_stream` | `xfusion_log_stream` | `xfusion-log-stream` |

---

## Best Practices Applied

**Implicit dependency declaration via attribute reference**

The `log_group_name` on the Log Stream is set using `aws_cloudwatch_log_group.xfusion_log_group.name` rather than a hardcoded string. This approach binds the two resources at the Terraform graph level, ensuring correct provisioning order and preventing orphaned stream creation attempts if the log group name ever changes.

**Single-file resource consolidation**

Both resources are defined in a single `main.tf` file, keeping the configuration surface minimal and reducing cognitive overhead for consumers of the codebase. This is appropriate for tightly coupled resource pairs of this scale.

**Heredoc authoring for file creation**

Using `cat > main.tf << 'EOF' ... EOF` with a quoted delimiter prevents shell variable interpolation inside the heredoc block, which is critical when the file content itself contains characters like `$` or `{` that could be misinterpreted by the shell.

**Pre-apply plan review**

`terraform plan` was executed before `terraform apply` to inspect the full execution plan and confirm the expected diff (2 to add, 0 to change, 0 to destroy) before committing changes. This is a non-negotiable step in any production workflow.

**Post-apply state verification**

`terraform state list` was executed after apply to confirm that both resources were successfully registered in the state file. This closes the validation loop and ensures the infrastructure is fully under Terraform management.

**`terraform validate` before plan**

The configuration was validated before planning. Running `terraform validate` catches schema violations and syntax errors early, before provider initialization is required to evaluate the plan.

---

## Errors and Resolutions

No errors were encountered during this implementation. All five phases completed successfully on the first attempt.

**Potential failure modes and mitigations:**

| Scenario | Root Cause | Resolution |
|---|---|---|
| `terraform init` fails to download provider | Network isolation or incorrect provider source | Verify connectivity to the Terraform registry; confirm the `source` and `version` values in `provider.tf` match an available release |
| `apply` fails with "ResourceNotFoundException" on Log Stream | Log Group not yet available when stream creation is attempted | Ensure `log_group_name` is set via attribute reference, not a hardcoded string, to enforce dependency ordering |
| `apply` fails with endpoint connection error | LocalStack not running or endpoint URL misconfigured | Verify LocalStack is healthy and the `cloudwatch` endpoint in `provider.tf` resolves correctly |
| Heredoc write produces malformed file | Shell variable expansion inside the heredoc | Always use a quoted delimiter (`<< 'EOF'`) when the file content contains special characters |

---

## Lessons Learned

**Attribute references are preferable to hardcoded values for resource linkage**

Using `aws_cloudwatch_log_group.xfusion_log_group.name` to populate `log_group_name` on the Log Stream is strictly superior to hardcoding the string `"xfusion-log-group"`. The attribute reference encodes the dependency in the Terraform resource graph, which controls provisioning order and propagates changes automatically during future modifications.

**Validate early in every Terraform workflow**

Running `terraform validate` immediately after authoring catches errors at the cheapest possible point in the lifecycle, before providers need to be initialized and before any API calls are made. Making this a habitual step reduces debugging cycles significantly.

**`terraform plan` output is the authoritative pre-change audit record**

The plan output explicitly confirms the number of resources to be added, changed, and destroyed. Verifying that the plan shows exactly `2 to add, 0 to change, 0 to destroy` before approving the apply is a form of pre-execution safety check that mirrors code review in software development workflows.

**LocalStack endpoint configuration is an all-or-nothing prerequisite**

The `provider.tf` must include the `cloudwatch` endpoint override for the `aws_cloudwatch_log_group` and `aws_cloudwatch_log_stream` resources to resolve correctly against LocalStack. A missing or misconfigured endpoint entry would cause the apply to fail silently or route to the real AWS API, both of which are unacceptable outcomes in a local development environment.

**Post-apply state verification closes the delivery loop**

Running `terraform state list` after every apply confirms that the state file reflects the intended resource inventory. A successful `apply` output alone is insufficient; the state file is the ground truth for what Terraform manages, and confirming its contents is the final step in responsible infrastructure delivery.

---

## References

* [Terraform AWS Provider: aws_cloudwatch_log_group](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_group)
* [Terraform AWS Provider: aws_cloudwatch_log_stream](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudwatch_log_stream)
* [Terraform CLI: init](https://developer.hashicorp.com/terraform/cli/commands/init)
* [Terraform CLI: validate](https://developer.hashicorp.com/terraform/cli/commands/validate)
* [Terraform CLI: plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
* [Terraform CLI: apply](https://developer.hashicorp.com/terraform/cli/commands/apply)
* [Terraform CLI: state list](https://developer.hashicorp.com/terraform/cli/commands/state/list)
* [LocalStack Documentation](https://docs.localstack.cloud/overview/)





<img width="1046" height="736" alt="image" src="https://github.com/user-attachments/assets/3abae4d4-f927-4a0a-ac6f-6905e751ce1a" />
<img width="1044" height="381" alt="image" src="https://github.com/user-attachments/assets/d652faf2-625f-4bf0-be6d-9246b0cddd5f" />
