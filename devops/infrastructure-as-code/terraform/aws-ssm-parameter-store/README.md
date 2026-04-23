# Terraform AWS SSM Parameter Provisioning via LocalStack

> Provisioning an AWS Systems Manager (SSM) Parameter Store entry using Terraform against a LocalStack endpoint, with full lifecycle validation via the AWS CLI.

---

## Table of Contents

* [Overview](#overview)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Check Terraform Version](#step-1-check-terraform-version)
  * [Step 2: List Working Directory Contents](#step-2-list-working-directory-contents)
  * [Step 3: Check AWS CLI Version](#step-3-check-aws-cli-version)
  * [Step 4: Review the Provider Configuration](#step-4-review-the-provider-configuration)
  * [Step 5: Author the SSM Parameter Resource](#step-5-author-the-ssm-parameter-resource)
  * [Step 6: Initialize the Terraform Working Directory](#step-6-initialize-the-terraform-working-directory)
  * [Step 7: Confirm main.tf Contents After Init](#step-7-confirm-maintf-contents-after-init)
  * [Step 8: Validate the Configuration](#step-8-validate-the-configuration)
  * [Step 9: Apply the Configuration](#step-9-apply-the-configuration)
  * [Step 10: Verify the Parameter via AWS CLI](#step-10-verify-the-parameter-via-aws-cli)
* [Resource Specification](#resource-specification)
* [Best Practices Applied](#best-practices-applied)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)

---

## Overview

This implementation provisions a named AWS Systems Manager Parameter Store entry using Terraform as the infrastructure-as-code engine. The target environment is a LocalStack instance, which emulates AWS service endpoints locally, enabling full IaC lifecycle testing without incurring cloud costs or requiring live AWS credentials.

The parameter is of type `String`, carries a defined value, and is scoped to the `us-east-1` region. Upon successful apply, the parameter is retrieved using the AWS CLI to confirm end-to-end provisioning fidelity.

---

## Architecture and Design Intent

```
+---------------------------+
|   Terraform CLI (v1.11.0) |
|   main.tf + provider.tf   |
+-----------+---------------+
            |
            | AWS Provider v5.91.0
            |
+-----------v---------------------+
|   LocalStack (http://aws:4566)  |
|   AWS SSM Parameter Store       |
|   Region: us-east-1             |
+---------------------------------+
            |
            | AWS CLI verification
            |
+-----------v---------------------+
|   aws ssm get-parameter         |
|   --endpoint-url http://aws:4566|
+---------------------------------+
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
| OS | Linux/amd64 (6.8.0-106-generic) |
| LocalStack | Running and accessible at `http://aws:4566` |

The Terraform working directory is `/home/bob/terraform`. Only `main.tf` is authored during this implementation. The `provider.tf` file is pre-existing and must not be modified.

---

## Repository Structure

```
/home/bob/terraform/
├── provider.tf           # Pre-configured AWS provider with LocalStack endpoints
├── main.tf               # SSM Parameter resource definition (authored in this task)
├── README.MD             # Original task description
├── .terraform/           # Generated after terraform init
└── .terraform.lock.hcl   # Provider dependency lock file (generated after terraform init)
```

---

## Implementation Guide

### Step 1: Check Terraform Version

Confirm the installed Terraform version before beginning any configuration work.

```bash
terraform version
```

**Output:**

```
Terraform v1.11.0
on linux_amd64

Your version of Terraform is out of date! The latest version
is 1.14.9. You can update by downloading from https://www.terraform.io/downloads.html
```

> Screenshot: Terraform version output showing v1.11.0 on linux_amd64 with upgrade notice


<img width="1051" height="604" alt="image" src="https://github.com/user-attachments/assets/f7ff81c8-1a5b-4306-bdd3-734f0b56ae8e" />

The upgrade notice is informational only and does not block execution. Terraform v1.11.0 is fully functional for this implementation.

---

### Step 2: List Working Directory Contents

Inspect the contents of the Terraform working directory to understand what files are already present before creating any new configuration.

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

> Screenshot: Working directory listing showing only README.MD and provider.tf present

<img width="1051" height="604" alt="image" src="https://github.com/user-attachments/assets/f7ff81c8-1a5b-4306-bdd3-734f0b56ae8e" />

Only `README.MD` and `provider.tf` exist at this point. The `main.tf` file does not yet exist and will be created in a later step.

---

### Step 3: Check AWS CLI Version

Confirm the AWS CLI is available and note its version for traceability.

```bash
aws --version
```

**Output:**

```
aws-cli/1.40.38 Python/3.10.12 Linux/6.8.0-106-generic botocore/1.38.39
```

> Screenshot: AWS CLI version output confirming availability and version details

<img width="1049" height="659" alt="image" src="https://github.com/user-attachments/assets/88c1b577-01b6-4950-9c94-0e01a30f0749" />

---

### Step 4: Review the Provider Configuration

Inspect the pre-existing `provider.tf` to understand how the AWS provider is configured, including endpoint routing and authentication bypass settings, before authoring any resource files.

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

> Screenshot: Full contents of provider.tf displayed in terminal


<img width="1082" height="643" alt="image" src="https://github.com/user-attachments/assets/19b17966-9f28-40eb-9701-36564fdb0327" />

Key observations:
* The AWS provider is pinned to version `5.91.0`.
* The `ssm` endpoint is explicitly mapped to `http://aws:4566`, confirming all SSM API calls route through LocalStack.
* `skip_credentials_validation = true` and `skip_requesting_account_id = true` disable real AWS authentication checks, which are not applicable in a LocalStack environment.
* `s3_use_path_style = true` enables path-style S3 addressing as required by LocalStack.

---

### Step 5: Author the SSM Parameter Resource

Create `main.tf` in the Terraform working directory using a heredoc to write the resource configuration. The single quotes around `EOF` (`<< 'EOF'`) prevent shell variable interpolation, ensuring the file content is written exactly as typed.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_ssm_parameter" "nautilus_param" {
  name  = "nautilus-ssm-parameter"
  type  = "String"
  value = "nautilus-value"
}
EOF
```

> Screenshot: Heredoc command writing the aws_ssm_parameter resource block into main.tf

<img width="1033" height="615" alt="image" src="https://github.com/user-attachments/assets/575354c2-d832-4928-ae89-a48be0f0b1f2" />

---

### Step 6: Initialize the Terraform Working Directory

Run `terraform init` to download the pinned AWS provider version specified in `provider.tf` and prepare the working directory for plan and apply operations.

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

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration, rerun this command
to reinitialize your working directory. If you forget, other commands will
detect it and remind you to do so if necessary.
```

> Screenshot: terraform init output confirming provider download, lock file creation, and successful initialization

<img width="1047" height="747" alt="image" src="https://github.com/user-attachments/assets/bccf962b-f291-45b4-8f22-63d5af40ce80" />

The `.terraform.lock.hcl` file is created during this step, locking the provider version for reproducible runs across environments.

---

### Step 7: Confirm main.tf Contents After Init

Read back the contents of `main.tf` to confirm the resource block was persisted correctly before proceeding to validation and apply.

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

> Screenshot: cat main.tf output confirming the resource block is intact after terraform init

<img width="1054" height="673" alt="image" src="https://github.com/user-attachments/assets/1cc7b837-5a72-4ba8-a5cf-d125cc4770a8" />

The file contents match exactly what was written in Step 5. All three required attributes (`name`, `type`, `value`) are present and correctly set.

---

### Step 8: Validate the Configuration

Run `terraform validate` to perform a static syntax and schema check across all `.tf` files in the working directory before executing any apply.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

> Screenshot: terraform validate success message confirming the configuration is syntactically and schema-valid

<img width="1052" height="614" alt="image" src="https://github.com/user-attachments/assets/e8410197-a141-4ea1-b8e2-3a39262750b8" />

Validation confirms the HCL syntax is well-formed and all resource attributes are valid according to the AWS provider schema.

---

### Step 9: Apply the Configuration

Execute `terraform apply` with the `-auto-approve` flag to provision the SSM parameter without requiring interactive confirmation. Terraform displays the full execution plan before creating the resource.

```bash
terraform apply -auto-approve
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
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

> Screenshot: terraform apply output showing the execution plan, resource creation progress, and apply complete summary

<img width="1047" height="653" alt="image" src="https://github.com/user-attachments/assets/b749200c-bc04-49fe-8f9d-d7fbc5ad722b" />

Key observations from the apply output:
* `value` is rendered as `(sensitive value)` in the plan. This is the default behavior for SSM parameter values in the Terraform AWS provider and is correct production behavior.
* `value_wo` appears as a write-only attribute, a provider feature that allows secure value injection without storing the value in Terraform state.
* The resource ID is assigned as `nautilus-ssm-parameter`, matching the `name` attribute defined in `main.tf`.
* Creation completed in 0 seconds, as expected for a LocalStack-backed resource.

---

### Step 10: Verify the Parameter via AWS CLI

Retrieve the provisioned parameter using the AWS CLI, targeting the LocalStack endpoint explicitly to confirm the parameter exists and holds the correct values.

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

> Screenshot: AWS CLI get-parameter JSON response confirming Name, Type, Value, Version, ARN, and DataType

<img width="1050" height="439" alt="image" src="https://github.com/user-attachments/assets/3656535c-daa2-4d92-88f0-0b603ffa3a12" />


**Verification checklist:**

| Field | Expected | Returned |
|---|---|---|
| Name | `nautilus-ssm-parameter` | `nautilus-ssm-parameter` |
| Type | `String` | `String` |
| Value | `nautilus-value` | `nautilus-value` |
| Version | `1` | `1` |
| Region (from ARN) | `us-east-1` | `us-east-1` |
| DataType | `text` | `text` |

The account ID in the ARN (`000000000000`) is the standard placeholder used by LocalStack for all emulated AWS resources.

---

## Resource Specification

```hcl
resource "aws_ssm_parameter" "nautilus_param" {
  name  = "nautilus-ssm-parameter"
  type  = "String"
  value = "nautilus-value"
}
```

| Attribute | Value |
|---|---|
| Terraform resource address | `aws_ssm_parameter.nautilus_param` |
| SSM parameter name | `nautilus-ssm-parameter` |
| Type | `String` |
| Value | `nautilus-value` |
| Tier | Standard (provider default) |
| Encryption | None (plaintext String) |
| Region | `us-east-1` |

---

## Best Practices Applied

* **Read before writing:** The working directory was listed and `provider.tf` was inspected before creating any new files, confirming the endpoint configuration and preventing conflicts with pre-existing infrastructure code.

* **Heredoc with quoted delimiter:** Using `<< 'EOF'` (single-quoted) for file creation prevents shell variable interpolation, ensuring HCL content is written to disk verbatim without unintended substitutions.

* **Confirm file contents at a natural checkpoint:** Running `cat main.tf` after `terraform init` confirms the file persisted correctly at the point where it matters most, immediately before validation and apply.

* **Validate before apply:** Running `terraform validate` before `terraform apply` catches syntax and schema errors without touching remote state, preventing partial provisioning from a malformed configuration.

* **Pinned provider version:** The AWS provider is pinned to `5.91.0` in `provider.tf` and locked in `.terraform.lock.hcl`, ensuring deterministic behavior across all environments and preventing unintended provider upgrades.

* **Independent CLI verification:** Using `aws ssm get-parameter` to verify the parameter decouples the verification from Terraform state, confirming that the actual LocalStack API endpoint reflects the provisioned state and not just Terraform's internal record.

* **Explicit `--endpoint-url` and `--region` on CLI commands:** Specifying these flags on every AWS CLI command in a LocalStack environment prevents accidental API calls to real AWS and makes the target endpoint unambiguous.

* **`--output json` for structured verification:** JSON output makes the CLI verification result parseable, unambiguous, and suitable for integration into automated acceptance pipelines.

---

## Errors and Resolutions

No errors were encountered during this implementation. The execution proceeded cleanly through all ten steps from version checks through CLI verification.

**Anticipated failure modes in similar environments and their resolutions:**

| Potential Error | Root Cause | Resolution |
|---|---|---|
| `connection refused` on `terraform apply` | LocalStack not running or the `aws` hostname not resolving to the container IP | Confirm the LocalStack container is running and hostname resolves correctly |
| `Error: Invalid provider configuration` | Missing or malformed `endpoints` block in `provider.tf` | Confirm the `ssm` key is present in the `endpoints` block pointing to `http://aws:4566` |
| `Error: InvalidKeyId` | Using a KMS key reference with a `SecureString` type in LocalStack without KMS emulation enabled | Use `type = "String"` for LocalStack environments unless KMS emulation is explicitly configured |
| Terraform upgrade warning on `terraform version` | Running v1.11.0 while v1.14.9 is the latest stable release | Warning is non-blocking; upgrade during a scheduled maintenance window |
| `ResourceNotFoundException` on `aws ssm get-parameter` | CLI targeting wrong endpoint or region | Always pass `--endpoint-url http://aws:4566` and `--region us-east-1` explicitly on every command |

---

## Lessons Learned

* **The exact order of operations matters for reproducibility:** In this implementation, `cat main.tf` was run after `terraform init`, not immediately after the heredoc write. Documenting the exact sequence preserves full reproducibility for anyone following this runbook in a new environment.

* **LocalStack account ID is always `000000000000`:** ARNs returned by LocalStack use this placeholder. Any automation or validation script that parses ARNs must account for this rather than expecting a real 12-digit AWS account number.

* **`value` is masked as sensitive by default in SSM resources:** The Terraform AWS provider suppresses the `value` attribute in plan output to protect it from being exposed in logs. The actual value must always be verified through an independent API call, as demonstrated in Step 10.

* **`value_wo` signals write-only state handling:** The presence of `value_wo` in the plan output indicates the provider supports write-only attribute injection, preventing sensitive values from being persisted in the Terraform state file. In production environments, this should be preferred over `value` for `SecureString` parameters.

* **`terraform init` must precede `terraform validate`:** The validate command requires the provider plugin to be installed locally. The correct mandatory sequence is always: init, validate, apply.

* **Explicit endpoint mapping prevents silent real-AWS calls:** Defining every service endpoint individually in the provider block makes it immediately clear which services are emulated. Adding a new resource type without a corresponding endpoint entry in `provider.tf` will fail fast rather than silently routing to real AWS infrastructure.
