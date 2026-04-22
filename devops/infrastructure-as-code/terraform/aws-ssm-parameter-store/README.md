# Terraform AWS SSM Parameter Provisioning via LocalStack

> Provisioning an AWS Systems Manager (SSM) Parameter Store entry using Terraform against a LocalStack endpoint, with full lifecycle validation via the AWS CLI.

---

## Table of Contents

* [Overview](#overview)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Toolchain Versions](#step-1-verify-toolchain-versions)
  * [Step 2: Inspect the Working Directory](#step-2-inspect-the-working-directory)
  * [Step 3: Review the Provider Configuration](#step-3-review-the-provider-configuration)
  * [Step 4: Author the SSM Parameter Resource](#step-4-author-the-ssm-parameter-resource)
  * [Step 5: Initialize the Terraform Working Directory](#step-5-initialize-the-terraform-working-directory)
  * [Step 6: Validate the Configuration](#step-6-validate-the-configuration)
  * [Step 7: Apply the Configuration](#step-7-apply-the-configuration)
  * [Step 8: Verify the Parameter via AWS CLI](#step-8-verify-the-parameter-via-aws-cli)
* [Resource Specification](#resource-specification)
* [Best Practices Applied](#best-practices-applied)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)

---

## Overview

This implementation provisions a named AWS Systems Manager Parameter Store entry using Terraform as the infrastructure-as-code engine. The target environment is a LocalStack instance, which emulates AWS service endpoints locally, enabling full IaC lifecycle testing without incurring cloud costs or requiring live AWS credentials.

The parameter is of type `String`, carries a known value, and is scoped to the `us-east-1` region. Upon successful apply, the parameter is retrieved using the AWS CLI to confirm end-to-end provisioning fidelity.

---

## Architecture and Design Intent

```
+--------------------------+
|   Terraform CLI (v1.11)  |
|   main.tf + provider.tf  |
+-----------+--------------+
            |
            | Terraform AWS Provider v5.91.0
            |
+-----------v------------------+
|   LocalStack (http://aws:4566)|
|   AWS SSM Parameter Store    |
|   Region: us-east-1          |
+------------------------------+
            |
            | AWS CLI verification
            |
+-----------v------------------+
|  aws ssm get-parameter       |
|  --endpoint-url http://aws:4566 |
+------------------------------+
```

All AWS API calls are routed through LocalStack's unified endpoint at `http://aws:4566`. The Terraform provider is configured with `skip_credentials_validation = true` and `skip_requesting_account_id = true` to bypass authentication checks that are irrelevant in a local emulation context.

---

## Prerequisites

| Requirement | Version Used |
|---|---|
| Terraform | v1.11.0 |
| AWS CLI | v1.40.38 |
| Python | 3.10.12 |
| botocore | 1.38.39 |
| LocalStack | Running at `http://aws:4566` |
| OS | Ubuntu (Linux x86_64) |

The Terraform working directory is `/home/bob/terraform`. All files must be created within this directory. Only `main.tf` is authored during this implementation; `provider.tf` is pre-existing.

---

## Repository Structure

```
/home/bob/terraform/
├── provider.tf          # Pre-configured AWS provider with LocalStack endpoints
├── main.tf              # SSM Parameter resource definition (authored in this task)
├── README.MD            # Original task description
├── .terraform/          # Generated after terraform init
│   └── providers/...
└── .terraform.lock.hcl  # Provider dependency lock file
```

---

## Implementation Guide

### Step 1: Verify Toolchain Versions

Confirm that Terraform and the AWS CLI are available and note the active versions before beginning any configuration work.

```bash
terraform version
aws --version
```

**Output:**

```
Terraform v1.11.0
on linux_amd64

Your version of Terraform is out of date! The latest version
is 1.14.9. You can update by downloading from https://www.terraform.io/downloads.html

aws-cli/1.40.38 Python/3.10.12 Linux/6.8.0-106-generic botocore/1.38.39
```

> Screenshot: Terraform and AWS CLI version output in terminal

The version mismatch warning from Terraform is expected and non-blocking in this context. Terraform v1.11.0 is fully compatible with AWS provider v5.91.0.

---

### Step 2: Inspect the Working Directory

List the contents of the working directory to understand what is already present before authoring new configuration files.

```bash
ls -la
```

**Output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 22 23:34 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> Screenshot: Working directory listing showing pre-existing provider.tf and README.MD

Only `provider.tf` and `README.MD` are present. `main.tf` does not yet exist and must be created.

---

### Step 3: Review the Provider Configuration

Inspect the existing `provider.tf` to understand endpoint routing and provider constraints before writing any resource configuration.

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

> Screenshot: Contents of provider.tf displayed in terminal

Key observations:
* The `ssm` endpoint is explicitly mapped to `http://aws:4566`, confirming that SSM API calls will route through LocalStack.
* `skip_credentials_validation` and `skip_requesting_account_id` disable credential and account checks that would fail against LocalStack.
* `s3_use_path_style` enables path-style S3 addressing, required by LocalStack.

---

### Step 4: Author the SSM Parameter Resource

Create `main.tf` with a single `aws_ssm_parameter` resource block. The heredoc syntax (`<< 'EOF'`) is used to write the file contents directly from the terminal, ensuring no trailing whitespace or encoding issues.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_ssm_parameter" "nautilus_param" {
  name  = "nautilus-ssm-parameter"
  type  = "String"
  value = "nautilus-value"
}
EOF
```

Confirm the file was written correctly:

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_ssm_parameter" "nautilus_param" {
  name  = "nautilus-ssm-parameter"
  type  = "String"
  value = "nautilus-value"
}
```

> Screenshot: main.tf contents confirmed in terminal after heredoc write

**Resource attributes:**

| Attribute | Value |
|---|---|
| Resource type | `aws_ssm_parameter` |
| Terraform resource name | `nautilus_param` |
| SSM parameter name | `nautilus-ssm-parameter` |
| Type | `String` |
| Value | `nautilus-value` |

---

### Step 5: Initialize the Terraform Working Directory

Run `terraform init` to download the pinned AWS provider version and prepare the working directory for plan and apply operations.

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

> Screenshot: terraform init output showing provider download and lock file creation

The `.terraform.lock.hcl` file is generated during this step, pinning the exact provider version for reproducible runs across environments.

---

### Step 6: Validate the Configuration

Run `terraform validate` to perform a static syntax and schema check on all `.tf` files in the working directory before executing any plan or apply.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

> Screenshot: terraform validate success message in terminal

Validation confirms that the HCL syntax is well-formed and that all referenced resource attributes are valid according to the AWS provider schema.

---

### Step 7: Apply the Configuration

Execute `terraform apply` with the `-auto-approve` flag to provision the SSM parameter without requiring interactive confirmation. Review the execution plan displayed before the apply begins.

```bash
terraform apply -auto-approve
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ssm_parameter.nautilus_param will be created
  + resource "aws_ssm_parameter" "nautilus_param" {
      + arn            = (known after apply)
      + data_type      = (known after apply)
      + has_value_wo   = (known after apply)
      + id             = (known after apply)
      + insecure_value = (known after apply)
      + key_id         = (known after apply)
      + name           = "nautilus-ssm-parameter"
      + tags_all       = (known after apply)
      + tier           = (known after apply)
      + type           = "String"
      + value          = (sensitive value)
      + value_wo       = (write-only attribute)
      + version        = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ssm_parameter.nautilus_param: Creating...
aws_ssm_parameter.nautilus_param: Creation complete after 0s [id=nautilus-ssm-parameter]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> Screenshot: terraform apply output showing plan, creation, and apply complete summary

Key observations from the apply output:
* The `value` attribute is rendered as `(sensitive value)` in the plan, which is the default behavior for SSM parameter values in Terraform. This protects the value from being exposed in logs or terminal output.
* `value_wo` is a write-only attribute introduced in newer provider versions, enabling secure value injection without storing the value in state.
* The resource ID is set to `nautilus-ssm-parameter`, which is the parameter name as expected.

---

### Step 8: Verify the Parameter via AWS CLI

Retrieve the newly created parameter using the AWS CLI, targeting the LocalStack endpoint directly to confirm the parameter exists and holds the correct value.

```bash
aws ssm get-parameter \
  --name "nautilus-ssm-parameter" \
  --endpoint-url http://aws:4566 \
  --region us-east-1 \
  --output json
```

**Output:**

```json
{
    "Parameter": {
        "Name": "nautilus-ssm-parameter",
        "Type": "String",
        "Value": "nautilus-value",
        "Version": 1,
        "LastModifiedDate": 1776901225.156,
        "ARN": "arn:aws:ssm:us-east-1:000000000000:parameter/nautilus-ssm-parameter",
        "DataType": "text"
    }
}
```

> Screenshot: AWS CLI get-parameter JSON response confirming parameter name, type, value, and ARN

**Verification checklist:**

| Field | Expected | Confirmed |
|---|---|---|
| Name | `nautilus-ssm-parameter` | `nautilus-ssm-parameter` |
| Type | `String` | `String` |
| Value | `nautilus-value` | `nautilus-value` |
| Region (ARN) | `us-east-1` | `us-east-1` |
| Version | 1 (first write) | 1 |
| DataType | `text` | `text` |

The account ID in the ARN is `000000000000`, which is the placeholder account ID used by LocalStack for all emulated resources.

---

## Resource Specification

```hcl
resource "aws_ssm_parameter" "nautilus_param" {
  name  = "nautilus-ssm-parameter"
  type  = "String"
  value = "nautilus-value"
}
```

| Parameter | Detail |
|---|---|
| Terraform resource address | `aws_ssm_parameter.nautilus_param` |
| SSM path | `/nautilus-ssm-parameter` |
| Tier | Standard (default) |
| Encryption | None (plaintext String) |
| Region | `us-east-1` |

---

## Best Practices Applied

* **Single responsibility per file:** Resource definitions are isolated in `main.tf`, keeping provider configuration cleanly separated in `provider.tf`. This separation improves maintainability and allows provider changes without touching resource logic.

* **Heredoc for file creation:** Using `<< 'EOF'` heredoc syntax prevents shell variable interpolation and ensures the file content is written verbatim, which is critical for HCL that may contain `$` characters.

* **Validate before apply:** Running `terraform validate` before `terraform apply` catches syntax and schema errors early, avoiding partial state corruption that can occur if an apply fails mid-execution.

* **Pinned provider version:** The AWS provider is pinned to `5.91.0` in `provider.tf` and locked in `.terraform.lock.hcl`. This ensures deterministic behavior across all environments and prevents unintended provider upgrades from introducing breaking changes.

* **CLI verification after apply:** Independently verifying the resource via `aws ssm get-parameter` decouples the verification from Terraform state, confirming that the actual API endpoint reflects the provisioned state and not just Terraform's internal record.

* **Explicit endpoint mapping:** All AWS service endpoints are explicitly mapped in the provider block, eliminating ambiguity about which service calls are routed to LocalStack and which might inadvertently escape to real AWS.

* **`--output json` on CLI verification:** Using structured JSON output makes the verification result unambiguous, parseable, and easy to integrate into automated acceptance testing pipelines.

---

## Errors and Resolutions

No errors were encountered during this implementation. The execution proceeded cleanly from init through apply and CLI verification.

**Anticipated issues in similar environments and their mitigations:**

| Potential Error | Root Cause | Resolution |
|---|---|---|
| `connection refused` on `terraform apply` | LocalStack not running or DNS resolution for `aws` hostname failing | Confirm LocalStack container is running and `aws` hostname resolves to the LocalStack container IP |
| `Error: Invalid provider configuration` | Missing or malformed `endpoints` block in `provider.tf` | Ensure the `ssm` key is present in the `endpoints` block pointing to `http://aws:4566` |
| `Error: InvalidKeyId` | Attempting to use a KMS key with a `SecureString` parameter in LocalStack | Use `type = "String"` for LocalStack environments unless the LocalStack tier supports KMS |
| Terraform version warning on `terraform version` | Running Terraform v1.11.0 while v1.14.9 is available | Warning is non-blocking; upgrade at a scheduled maintenance window |

---

## Lessons Learned

* **LocalStack account ID is always `000000000000`:** When parsing ARNs from LocalStack responses, expect this placeholder account ID rather than a real 12-digit AWS account number. Automation scripts that validate ARN structure must account for this.

* **`value` is treated as sensitive by default in SSM resources:** The Terraform AWS provider marks the `value` attribute of `aws_ssm_parameter` as sensitive, which masks it in plan output. This is correct production behavior and should not be disabled. Retrieve the actual value through the AWS CLI or SSM console, never from Terraform plan logs.

* **`write-only` attributes signal provider maturity improvements:** The presence of `value_wo` in the plan output indicates the provider supports write-only value injection, a feature introduced to prevent sensitive parameter values from being stored in Terraform state files. In production, prefer `value_wo` over `value` for `SecureString` parameters.

* **Provider initialization is environment-specific:** The `.terraform` directory and `.terraform.lock.hcl` are generated locally and should be committed to version control for team consistency, but the `.terraform/` directory itself should be added to `.gitignore` to avoid committing binary provider plugins.

* **Explicit endpoint specification scales to multi-service architectures:** Defining each AWS service endpoint individually in the provider block, rather than relying on a wildcard override, makes it immediately clear which services are emulated and prevents accidental API calls to real AWS when adding new resource types.


<img width="1051" height="604" alt="image" src="https://github.com/user-attachments/assets/f7ff81c8-1a5b-4306-bdd3-734f0b56ae8e" />
<img width="1049" height="659" alt="image" src="https://github.com/user-attachments/assets/88c1b577-01b6-4950-9c94-0e01a30f0749" />
<img width="1082" height="643" alt="image" src="https://github.com/user-attachments/assets/19b17966-9f28-40eb-9701-36564fdb0327" />
<img width="1033" height="615" alt="image" src="https://github.com/user-attachments/assets/575354c2-d832-4928-ae89-a48be0f0b1f2" />
<img width="1047" height="747" alt="image" src="https://github.com/user-attachments/assets/bccf962b-f291-45b4-8f22-63d5af40ce80" />
<img width="1054" height="673" alt="image" src="https://github.com/user-attachments/assets/1cc7b837-5a72-4ba8-a5cf-d125cc4770a8" />
<img width="1052" height="614" alt="image" src="https://github.com/user-attachments/assets/e8410197-a141-4ea1-b8e2-3a39262750b8" />
<img width="1047" height="653" alt="image" src="https://github.com/user-attachments/assets/b749200c-bc04-49fe-8f9d-d7fbc5ad722b" />
<img width="1050" height="439" alt="image" src="https://github.com/user-attachments/assets/3656535c-daa2-4d92-88f0-0b603ffa3a12" />

