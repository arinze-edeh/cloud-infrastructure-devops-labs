# AWS Kinesis Stream Provisioning with Terraform

Provisioning a real-time AWS Kinesis Data Stream using Terraform on a LocalStack-backed infrastructure environment. This implementation demonstrates Infrastructure as Code (IaC) best practices for streaming data pipeline provisioning, including idempotent configuration management, state-driven plan validation, and tag drift remediation.

---

## Table of Contents

* [Overview](#overview)
* [Architecture and Context](#architecture-and-context)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1 - Verify Working Directory](#step-1---verify-working-directory)
  * [Step 2 - Author the Terraform Configuration](#step-2---author-the-terraform-configuration)
  * [Step 3 - Initialize the Terraform Working Directory](#step-3---initialize-the-terraform-working-directory)
  * [Step 4 - Validate the Configuration](#step-4---validate-the-configuration)
  * [Step 5 - Preview the Execution Plan](#step-5---preview-the-execution-plan)
  * [Step 6 - Apply the Configuration](#step-6---apply-the-configuration)
  * [Step 7 - Detect and Resolve Tag Drift](#step-7---detect-and-resolve-tag-drift)
  * [Step 8 - Verify Idempotency](#step-8---verify-idempotency)
* [Errors and Resolutions](#errors-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Overview

| Attribute | Detail |
|---|---|
| **Cloud Provider** | AWS (LocalStack) |
| **Service** | Amazon Kinesis Data Streams |
| **IaC Tool** | Terraform v1.x with AWS Provider v5.91.0 |
| **Stream Name** | `datacenter-stream` |
| **Shard Count** | 1 |
| **Retention Period** | 24 hours |
| **Working Directory** | `/home/bob/terraform` |
| **Primary Config File** | `main.tf` |

---

## Architecture and Context

The Nautilus DevOps team requires a dedicated AWS Kinesis Data Stream to support real-time ingestion and processing of high-volume streaming data. This stream serves as a central pipeline between data-producing systems and downstream consumers that perform analytics and drive real-time decision-making.

Kinesis Data Streams provide a durable, ordered, and replayable message queue designed for high-throughput event-driven architectures. The stream is provisioned with a single shard sufficient for this workload and a 24-hour retention window to allow consumer replay in the event of downstream failures.

Terraform is the provisioning tool of choice, enabling declarative, version-controlled, and repeatable infrastructure management. All resources are defined exclusively in `main.tf` within the pre-existing working directory.

---

## Prerequisites

* Terraform CLI installed and available in `$PATH`
* AWS credentials configured (or LocalStack endpoint defined in `provider.tf`)
* An existing `provider.tf` file present in the working directory

---

## Repository Structure

```
/home/bob/terraform/
|-- .terraform/                  # Provider plugins (generated after init)
|-- .terraform.lock.hcl          # Provider version lock file (generated after init)
|-- provider.tf                  # AWS provider and backend configuration (pre-existing)
|-- main.tf                      # Kinesis stream resource definition (authored in this task)
|-- README.MD                    # Original task description
```

---

## Implementation Guide

### Step 1 - Verify Working Directory

Before authoring any configuration, confirm the working directory path and review existing files to understand the pre-configured state.

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
drwxr-x--- 1 bob bob 4096 Apr 20 02:34 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

The `provider.tf` file is pre-existing and contains the AWS provider configuration. Only `main.tf` is authored in this task; no additional `.tf` files are created.

> **Screenshot:** Working directory listing confirming `provider.tf` is present and no `main.tf` exists yet.

<img width="1050" height="656" alt="image" src="https://github.com/user-attachments/assets/1ac1eb84-77ea-44ee-9266-4bf024b41225" />

---

### Step 2 - Author the Terraform Configuration

Create `main.tf` using a heredoc to define the `aws_kinesis_stream` resource. The resource is named `datacenter_stream` internally and maps to the AWS resource name `datacenter-stream`.

```bash
cat > main.tf << 'EOF'
resource "aws_kinesis_stream" "datacenter_stream" {
  name             = "datacenter-stream"
  shard_count      = 1
  retention_period = 24

  tags = {
    Name = "datacenter-stream"
  }
}
EOF
```

Verify the file was written correctly:

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_kinesis_stream" "datacenter_stream" {
  name             = "datacenter-stream"
  shard_count      = 1
  retention_period = 24

  tags = {
    Name = "datacenter-stream"
  }
}
```

> **Screenshot:** Terminal output of `cat main.tf` confirming the resource block contents.

---

### Step 3 - Initialize the Terraform Working Directory

Run `terraform init` to download the AWS provider plugin specified in `provider.tf` and generate the lock file.

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

After initialization, verify the updated directory contents:

```bash
ls -la
```

**Output:**

```
total 32
drwxr-xr-x 1 bob bob 4096 Apr 20 02:41 .
drwxr-x--- 1 bob bob 4096 Apr 20 02:41 ..
drwxr-xr-x 3 bob bob 4096 Apr 20 02:41 .terraform
-rw-r--r-- 1 bob bob 1406 Apr 20 02:41 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  189 Apr 20 02:40 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> **Screenshot:** Terminal output of `terraform init` confirming provider installation and lock file creation.

---

### Step 4 - Validate the Configuration

Run `terraform validate` to perform a static syntax and schema check on the configuration before attempting a plan or apply.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

> **Screenshot:** Terminal output of `terraform validate` returning a success result.

---

### Step 5 - Preview the Execution Plan

Run `terraform plan` to review the proposed changes before committing them. This is a non-destructive, read-only operation.

```bash
terraform plan
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_kinesis_stream.datacenter_stream will be created
  + resource "aws_kinesis_stream" "datacenter_stream" {
      + arn                       = (known after apply)
      + encryption_type           = "NONE"
      + enforce_consumer_deletion = false
      + id                        = (known after apply)
      + name                      = "datacenter-stream"
      + retention_period          = 24
      + shard_count               = 1
      + tags                      = {
          + "Name" = "datacenter-stream"
        }
      + tags_all                  = {
          + "Name" = "datacenter-stream"
        }

      + stream_mode_details (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Terraform confirms one resource will be created with no changes or destructions.

> **Screenshot:** Terminal output of `terraform plan` showing the `+ create` action for `aws_kinesis_stream.datacenter_stream`.

---

### Step 6 - Apply the Configuration

Apply the configuration using the `-auto-approve` flag to bypass the interactive confirmation prompt. Terraform provisions the Kinesis stream and waits for it to reach the `ACTIVE` state.

```bash
terraform apply -auto-approve
```

**Output (truncated to key lines):**

```
aws_kinesis_stream.datacenter_stream: Creating...
aws_kinesis_stream.datacenter_stream: Still creating... [10s elapsed]
aws_kinesis_stream.datacenter_stream: Still creating... [20s elapsed]
aws_kinesis_stream.datacenter_stream: Creation complete after 21s [id=arn:aws:kinesis:us-east-1:000000000000:stream/datacenter-stream]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The stream is successfully provisioned and its ARN is captured in the Terraform state file.

> **Screenshot:** Terminal output of `terraform apply -auto-approve` confirming `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.`

---

### Step 7 - Detect and Resolve Tag Drift

After the initial apply, running `terraform plan` again revealed an unexpected in-place update due to tag drift. The remote resource returned the `Name` tag as absent from its current state, causing Terraform to detect a divergence and plan an update.

```bash
terraform plan
```

**Output showing drift:**

```
  # aws_kinesis_stream.datacenter_stream will be updated in-place
  ~ resource "aws_kinesis_stream" "datacenter_stream" {
      ~ tags = {
          + "Name" = "datacenter-stream"
        }
      ~ tags_all = {
          + "Name" = "datacenter-stream"
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

**Root Cause:** The LocalStack environment does not persist or return the `tags` block consistently in the `DescribeStreamSummary` response after creation, causing Terraform to evaluate the tag as missing and schedule a re-apply.

**Resolution:** Remove the `tags` block from `main.tf` entirely, since tags are not required by the task specification and their absence eliminates the drift detection loop. This keeps the configuration aligned with the actual remote state.

```bash
cat > main.tf << 'EOF'
resource "aws_kinesis_stream" "datacenter_stream" {
  name             = "datacenter-stream"
  shard_count      = 1
  retention_period = 24
}
EOF
```

Verify the updated file:

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_kinesis_stream" "datacenter_stream" {
  name             = "datacenter-stream"
  shard_count      = 1
  retention_period = 24
}
```

> **Screenshot:** Terminal output of `terraform plan` showing the tag drift `~ update in-place` action before the configuration was corrected.

> **Screenshot:** Terminal output of `cat main.tf` after the tags block was removed.

---

### Step 8 - Verify Idempotency

With the `tags` block removed, run `terraform plan` a final time to confirm the configuration is fully converged and no further changes are required.

```bash
terraform plan
```

**Output:**

```
aws_kinesis_stream.datacenter_stream: Refreshing state... [id=arn:aws:kinesis:us-east-1:000000000000:stream/datacenter-stream]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration and
found no differences, so no changes are needed.
```

The configuration is idempotent and the task is complete.

> **Screenshot:** Terminal output of final `terraform plan` returning `No changes. Your infrastructure matches the configuration.`

---

## Errors and Resolutions

### Tag Drift Causing Spurious In-Place Update

| Attribute | Detail |
|---|---|
| **Symptom** | `terraform plan` after a successful apply showed a pending `~ update in-place` modifying only the `tags` and `tags_all` attributes |
| **Root Cause** | LocalStack does not fully round-trip tag metadata in the Kinesis `DescribeStreamSummary` API response, so Terraform reads back an empty tag map and interprets the configured `Name` tag as a desired addition |
| **Resolution** | Removed the `tags` block from `main.tf` since tags were not required by the task. This aligned the configuration with what LocalStack persists, eliminating the drift signal |
| **Production Consideration** | In a real AWS environment, tags would persist correctly and this pattern would not occur. Tags should always be applied in production for cost allocation, access control, and governance |

---

## Best Practices Applied

* **Single-file discipline:** All resource configuration was contained within `main.tf` as specified. No additional `.tf` files were introduced, keeping the working directory minimal and predictable.

* **Validate before plan:** `terraform validate` was run before `terraform plan` to catch any syntax or schema errors early, before the provider attempted remote API calls.

* **Plan before apply:** A `terraform plan` was always executed before `terraform apply` to inspect the proposed change set and confirm no unintended mutations would occur.

* **Idempotency verification:** After every apply, `terraform plan` was re-run to confirm the state fully converged and no residual diff existed. A clean `No changes` output is the acceptance criterion for any Terraform deployment.

* **State-driven operations:** Terraform state was used as the source of truth for drift detection. The post-apply drift was identified through state refresh, not manual inspection, demonstrating the value of managed state.

* **Lock file committed:** The `.terraform.lock.hcl` file generated during `terraform init` pins the AWS provider to version `5.91.0`. This file should be committed to version control to guarantee reproducible provider installations across team members and CI pipelines.

* **Heredoc authoring:** The `cat > file << 'EOF'` pattern was used for clean, shell-safe file creation, preventing variable expansion and ensuring the written content exactly matches the intended configuration.

---

## Lessons Learned

* **LocalStack tag behavior differs from AWS:** When using LocalStack for local Terraform development, certain AWS API behaviors are simulated but not perfectly replicated. Tag persistence is one area where discrepancies can surface. Always verify idempotency with a second `terraform plan` after apply, and understand which gaps exist in the emulation layer being used.

* **A converged plan is the real deliverable:** The completion criterion for any Terraform task is not the `apply` itself but the subsequent `terraform plan` returning `No changes`. A successful apply followed by a pending update indicates an incomplete implementation.

* **Removing unneeded attributes prevents noise:** Adding optional attributes like `tags` that the underlying provider or environment cannot reliably persist creates configuration drift. Keeping the resource definition minimal and strictly aligned with requirements reduces operational noise.

* **`-auto-approve` is acceptable in controlled environments:** In a sandboxed or IaC testing environment where the plan has already been reviewed, `-auto-approve` is a reasonable choice. In production pipelines, approvals should be enforced through CI gate controls rather than omitted entirely.

* **ARN capture confirms provisioning success:** The ARN returned in the apply output (`arn:aws:kinesis:us-east-1:000000000000:stream/datacenter-stream`) confirms the resource was registered in the backend and its identity is now tracked in state. This ARN can be referenced by downstream resources using `aws_kinesis_stream.datacenter_stream.arn`.

---

## References

* [Terraform AWS Provider - aws_kinesis_stream](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/kinesis_stream)
* [Amazon Kinesis Data Streams Developer Guide](https://docs.aws.amazon.com/streams/latest/dev/introduction.html)
* [Terraform CLI - plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
* [Terraform CLI - apply](https://developer.hashicorp.com/terraform/cli/commands/apply)
* [Terraform CLI - validate](https://developer.hashicorp.com/terraform/cli/commands/validate)
* [LocalStack AWS Kinesis Simulation](https://docs.localstack.cloud/user-guide/aws/kinesis/)










<img width="1045" height="692" alt="image" src="https://github.com/user-attachments/assets/4a91be4f-58a5-462b-a645-eb577e39686a" />
<img width="1044" height="702" alt="image" src="https://github.com/user-attachments/assets/44879bf2-5468-4314-9dc5-75c91910faf1" />
<img width="1048" height="606" alt="image" src="https://github.com/user-attachments/assets/75470a3f-2db2-407e-99f8-5cd7ed1eb2cb" />
<img width="1046" height="587" alt="image" src="https://github.com/user-attachments/assets/78b28f96-1aa4-4aec-b05b-da4ce6a60c32" />
<img width="1052" height="657" alt="image" src="https://github.com/user-attachments/assets/1d1f5e69-8871-46ab-b567-e05fcd4782de" />
<img width="1042" height="719" alt="image" src="https://github.com/user-attachments/assets/7e2ad5e9-6498-45d2-9d40-535ca8234b62" />
<img width="1046" height="734" alt="image" src="https://github.com/user-attachments/assets/4d77a0b4-8572-4e27-a4ac-70dce3fe921d" />
<img width="1045" height="622" alt="image" src="https://github.com/user-attachments/assets/9296751b-90a6-4d9c-a812-e185ad59194d" />
<img width="1043" height="725" alt="image" src="https://github.com/user-attachments/assets/2e000131-5723-4f57-9ece-8b321ecf6776" />
<img width="1045" height="357" alt="image" src="https://github.com/user-attachments/assets/c584fc5a-ca38-4fb7-b9a8-b3c18c535da6" />
<img width="1050" height="430" alt="image" src="https://github.com/user-attachments/assets/2fdb89ba-ecf7-4f0b-99e1-b6c574247ce2" />
