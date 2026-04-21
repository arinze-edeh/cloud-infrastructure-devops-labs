# Terraform SNS Topic Provisioning on AWS (LocalStack)

Provision an AWS SNS topic named `xfusion-notifications` using Terraform against a LocalStack-backed AWS environment, working from a pre-configured provider file and authoring only a `main.tf` resource definition.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Inspect the Working Directory and Terraform Version](#step-1-inspect-the-working-directory-and-terraform-version)
  - [Step 2: Review the Existing Provider Configuration](#step-2-review-the-existing-provider-configuration)
  - [Step 3: Create main.tf with SNS Resource Only](#step-3-create-maintf-with-sns-resource-only)
  - [Step 4: Overwrite main.tf with a Full Provider Block and SNS Resource](#step-4-overwrite-maintf-with-a-full-provider-block-and-sns-resource)
  - [Step 5: Confirm main.tf Contents and Directory State](#step-5-confirm-maintf-contents-and-directory-state)
  - [Step 6: Run terraform init - Duplicate Provider Error](#step-6-run-terraform-init---duplicate-provider-error)
  - [Step 7: Fix main.tf by Removing the Provider Block](#step-7-fix-maintf-by-removing-the-provider-block)
  - [Step 8: Confirm Corrected main.tf Contents](#step-8-confirm-corrected-maintf-contents)
  - [Step 9: Re-run terraform init Successfully](#step-9-re-run-terraform-init-successfully)
  - [Step 10: Validate the Configuration](#step-10-validate-the-configuration)
  - [Step 11: Plan the Deployment](#step-11-plan-the-deployment)
  - [Step 12: Apply and Provision](#step-12-apply-and-provision)
  - [Step 13: Verify Terraform State](#step-13-verify-terraform-state)
  - [Step 14: Verify Resource via AWS CLI - localhost Endpoint Failure](#step-14-verify-resource-via-aws-cli---localhost-endpoint-failure)
  - [Step 15: Verify Resource via AWS CLI - Correct Endpoint](#step-15-verify-resource-via-aws-cli---correct-endpoint)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
  - [Error 1: Duplicate Provider Configuration on terraform init](#error-1-duplicate-provider-configuration-on-terraform-init)
  - [Error 2: AWS CLI Connection Failure Using localhost Endpoint](#error-2-aws-cli-connection-failure-using-localhost-endpoint)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)

---

## Overview

| Attribute | Value |
|---|---|
| **IaC Tool** | Terraform v1.11.0 |
| **Provider** | hashicorp/aws v5.91.0 |
| **Cloud Target** | LocalStack (http://aws:4566) |
| **Resource Provisioned** | AWS SNS Topic |
| **Topic Name** | xfusion-notifications |
| **Working Directory** | /home/bob/terraform |

The objective was to provision a single AWS SNS topic named `xfusion-notifications` using Terraform in a LocalStack-backed environment. A `provider.tf` file defining the AWS provider and all LocalStack endpoint overrides was already present in the working directory. The task required creating only a `main.tf` file to define the SNS resource without touching the existing provider configuration. Two errors were encountered and resolved during execution: a duplicate provider conflict from an incorrectly authored `main.tf`, and an AWS CLI endpoint mismatch during post-apply verification.

---

## Architecture and Design Intent

LocalStack simulates AWS service endpoints locally. All AWS API calls in this environment are routed to `http://aws:4566` rather than real AWS infrastructure. The `aws` hostname is the container network alias for the LocalStack service, not a localhost alias. The Terraform AWS provider is configured to skip credential validation and account ID resolution, which are not applicable in a local emulation context.

The SNS topic functions as a notification bus. In production, topics of this type underpin event-driven pipelines, alerting systems, and fan-out messaging architectures. This implementation reflects the minimal viable configuration for a standard, non-FIFO SNS topic.

The Terraform convention of separating provider configuration (`provider.tf`) from resource definitions (`main.tf`) improves maintainability, reduces the risk of duplicate provider conflicts, and enables cleaner modular reuse across environments.

---

## Prerequisites

* Terraform v1.11.0 or later installed
* AWS CLI installed and accessible in the shell
* LocalStack running and reachable at `http://aws:4566`
* Write access to `/home/bob/terraform`

---

## Repository Structure

```
/home/bob/terraform/
|-- provider.tf           # AWS provider and LocalStack endpoint configuration (pre-existing)
|-- main.tf               # SNS topic resource definition (created in this task)
|-- README.MD             # Original task description
|-- .terraform/           # Provider plugin cache (generated after terraform init)
|-- .terraform.lock.hcl   # Provider dependency lock file (generated after terraform init)
```

---

## Implementation Guide

### Step 1: Inspect the Working Directory and Terraform Version

List the contents of the working directory to understand what files already exist, then confirm the installed Terraform version.

```bash
ls -la
```

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 21 18:26 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

```bash
terraform version
```

```
Terraform v1.11.0
on linux_amd64

Your version of Terraform is out of date! The latest version
is 1.14.9. You can update by downloading from https://www.terraform.io/downloads.html
```

Two files are present: `README.MD` and `provider.tf`. No `main.tf` exists yet. Terraform v1.11.0 is installed. The version warning is informational only and does not affect the task.

*Screenshot: ls -la output showing README.MD and provider.tf, followed by terraform version output*

<img width="1048" height="512" alt="image" src="https://github.com/user-attachments/assets/ddaecc45-be28-41f7-84be-e2d63c8c8276" />

---

### Step 2: Review the Existing Provider Configuration

Read `provider.tf` to understand the AWS provider setup and LocalStack endpoint configuration before writing any new files.

```bash
cat /home/bob/terraform/provider.tf
```

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

The `provider "aws"` block is fully defined here, including `sns = "http://aws:4566"`. The `terraform` block also pins the provider version to `5.91.0`. This means `main.tf` must contain only the resource definition.

*Screenshot: cat output of provider.tf showing the full provider block and all LocalStack endpoint overrides*

<img width="1074" height="705" alt="image" src="https://github.com/user-attachments/assets/cb743707-be27-4ea2-8e45-92f3f9215254" />

---

### Step 3: Create main.tf with SNS Resource Only

Write the initial `main.tf` containing only the SNS topic resource block, with no provider configuration.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
EOF
```

*Screenshot: Terminal after the first cat heredoc write to main.tf*

<img width="1035" height="579" alt="image" src="https://github.com/user-attachments/assets/06be32cc-99e4-4189-a069-caf1eb6d8536" />

---

### Step 4: Overwrite main.tf with a Full Provider Block and SNS Resource

`main.tf` is overwritten a second time with a `cat` heredoc that now includes a complete `provider "aws"` block alongside the SNS resource. This version adds mock credentials and a `localhost`-based SNS endpoint directly into `main.tf`.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "mock_access_key"
  secret_key                  = "mock_secret_key"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    sns = "http://localhost:4566"
  }
}

resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
EOF
```

*Screenshot: Terminal after the second cat heredoc write, overwriting main.tf with the full provider block*

---

### Step 5: Confirm main.tf Contents and Directory State

Read the current contents of `main.tf` to confirm what was written, then list the directory to confirm both files are present.

```bash
cat /home/bob/terraform/main.tf
```

```hcl
provider "aws" {
  region                      = "us-east-1"
  access_key                  = "mock_access_key"
  secret_key                  = "mock_secret_key"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    sns = "http://localhost:4566"
  }
}

resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
```

```bash
ls -la /home/bob/terraform/
```

```
total 24
drwxr-xr-x 1 bob bob 4096 Apr 21 18:32 .
drwxr-x--- 1 bob bob 4096 Apr 21 18:29 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  414 Apr 21 18:33 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

`main.tf` now exists alongside `provider.tf`. At this point, both files define a `provider "aws"` block, which will cause a conflict on initialization.

*Screenshot: cat main.tf output showing the duplicate provider block, followed by ls -la confirming both .tf files exist*

---

### Step 6: Run terraform init - Duplicate Provider Error

Attempt to initialize the Terraform working directory. This fails because both `main.tf` and `provider.tf` now contain a default `provider "aws"` block.

```bash
terraform init
```

```
Initializing the backend...
╷
│ Error: Terraform encountered problems during initialisation, including problems
│ with the configuration, described below.
│
│ The Terraform configuration must be valid before initialization so that
│ Terraform can determine which modules and providers need to be installed.
│
╵
╷
│ Error: Duplicate provider configuration
│
│   on provider.tf line 10:
│   10: provider "aws" {
│
│ A default (non-aliased) provider configuration for "aws" was already given at main.tf:1,1-15. If multiple configurations are
│ required, set the "alias" argument for alternative configurations.
╵
╷
│ Error: Duplicate provider configuration
│
│   on provider.tf line 10:
│   10: provider "aws" {
│
│ A default (non-aliased) provider configuration for "aws" was already given at main.tf:1,1-15. If multiple configurations are
│ required, set the "alias" argument for alternative configurations.
╵
```

Terraform reports the error twice. The root cause is that `main.tf:1` and `provider.tf:10` each define a default (non-aliased) `provider "aws"` block. Terraform does not permit more than one default block for the same provider within a configuration directory. Initialization fails before any provider plugins are downloaded.

*Screenshot: terraform init output showing the duplicate provider configuration error reported twice*

---

### Step 7: Fix main.tf by Removing the Provider Block

Overwrite `main.tf` a third time, restoring it to contain only the SNS resource block with no provider configuration.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
EOF
```

*Screenshot: Terminal after the corrective cat heredoc write removing the provider block from main.tf*

---

### Step 8: Confirm Corrected main.tf Contents

Read `main.tf` to verify the provider block has been removed and only the resource definition remains.

```bash
cat /home/bob/terraform/main.tf
```

```hcl
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
```

`main.tf` now contains only the `aws_sns_topic` resource block. The provider conflict is resolved.

*Screenshot: cat main.tf output confirming resource-only content with no provider block*

---

### Step 9: Re-run terraform init Successfully

Initialize the Terraform working directory now that the duplicate provider conflict has been resolved.

```bash
terraform init
```

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

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.
```

The provider plugin `hashicorp/aws v5.91.0` is downloaded and installed successfully. A `.terraform.lock.hcl` file is generated to pin the provider version for reproducible runs.

*Screenshot: terraform init output confirming successful provider installation and lock file creation*

---

### Step 10: Validate the Configuration

Confirm that all Terraform configuration files are syntactically and semantically valid before planning.

```bash
terraform validate
```

```
Success! The configuration is valid.
```

*Screenshot: terraform validate output confirming zero configuration errors*

---

### Step 11: Plan the Deployment

Generate and review the execution plan to confirm the intended resource will be created with no unintended changes.

```bash
terraform plan
```

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_sns_topic.xfusion_notifications will be created
  + resource "aws_sns_topic" "xfusion_notifications" {
      + arn                         = (known after apply)
      + beginning_archive_time      = (known after apply)
      + content_based_deduplication = false
      + fifo_topic                  = false
      + id                          = (known after apply)
      + name                        = "xfusion-notifications"
      + name_prefix                 = (known after apply)
      + owner                       = (known after apply)
      + policy                      = (known after apply)
      + signature_version           = (known after apply)
      + tags_all                    = (known after apply)
      + tracing_config              = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

Note: You didn't use the -out option to save this plan, so Terraform can't guarantee to take exactly these actions if you run
"terraform apply" now.
```

The plan confirms one resource will be created: an SNS topic named `xfusion-notifications`. No existing state or infrastructure will be modified or destroyed. The `fifo_topic = false` attribute confirms this is a standard topic, not a FIFO topic.

*Screenshot: terraform plan output showing the complete aws_sns_topic resource attributes and the Plan: 1 to add summary*

---

### Step 12: Apply and Provision

Execute the plan and provision the SNS topic against the LocalStack endpoint.

```bash
terraform apply -auto-approve
```

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_sns_topic.xfusion_notifications will be created
  + resource "aws_sns_topic" "xfusion_notifications" {
      + arn                         = (known after apply)
      + beginning_archive_time      = (known after apply)
      + content_based_deduplication = false
      + fifo_topic                  = false
      + id                          = (known after apply)
      + name                        = "xfusion-notifications"
      + name_prefix                 = (known after apply)
      + owner                       = (known after apply)
      + policy                      = (known after apply)
      + signature_version           = (known after apply)
      + tags_all                    = (known after apply)
      + tracing_config              = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_sns_topic.xfusion_notifications: Creating...
aws_sns_topic.xfusion_notifications: Creation complete after 0s [id=arn:aws:sns:us-east-1:000000000000:xfusion-notifications]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The SNS topic is provisioned immediately. The assigned ARN `arn:aws:sns:us-east-1:000000000000:xfusion-notifications` uses the 12-zero account ID, which is the expected LocalStack behavior for simulated AWS account IDs.

*Screenshot: terraform apply -auto-approve output confirming resource creation and the assigned SNS topic ARN*

---

### Step 13: Verify Terraform State

Confirm that the provisioned resource is registered in Terraform state.

```bash
terraform state list
```

```
aws_sns_topic.xfusion_notifications
```

The SNS topic is tracked in state under the resource address `aws_sns_topic.xfusion_notifications`.

*Screenshot: terraform state list output confirming the SNS topic is present in Terraform state*

---

### Step 14: Verify Resource via AWS CLI - localhost Endpoint Failure

Attempt to list SNS topics using the `localhost` endpoint to confirm resource existence in LocalStack. This fails.

```bash
aws --endpoint-url=http://localhost:4566 sns list-topics
```

```
Could not connect to the endpoint URL: "http://localhost:4566/"
```

LocalStack is not reachable via `localhost` in this environment. The service is bound to the container network alias `aws`, not the loopback interface.

*Screenshot: AWS CLI error output showing connection failure to localhost:4566*

---

### Step 15: Verify Resource via AWS CLI - Correct Endpoint

Re-run the CLI verification using the correct `aws` hostname that maps to the LocalStack container.

```bash
aws --endpoint-url=http://aws:4566 sns list-topics
```

```json
{
    "Topics": [
        {
            "TopicArn": "arn:aws:sns:us-east-1:000000000000:xfusion-notifications"
        }
    ]
}
```

The SNS topic `xfusion-notifications` is confirmed to exist in LocalStack. The ARN matches what Terraform reported on apply, completing end-to-end verification.

*Screenshot: AWS CLI output listing the xfusion-notifications topic ARN from the correct aws:4566 endpoint*

---

## Errors Encountered and Resolutions

### Error 1: Duplicate Provider Configuration on terraform init

**What happened:**

After overwriting `main.tf` with a full `provider "aws"` block that included mock credentials and a `localhost`-based SNS endpoint, running `terraform init` failed. The error was reported twice:

```
Error: Duplicate provider configuration

  on provider.tf line 10:
  10: provider "aws" {

A default (non-aliased) provider configuration for "aws" was already given at main.tf:1,1-15. If multiple configurations are
required, set the "alias" argument for alternative configurations.
```

**Root cause:**

Terraform requires that each provider have at most one default (non-aliased) configuration block across all `.tf` files in the same working directory. `provider.tf` already contained `provider "aws" { ... }` at line 10. Writing a second `provider "aws"` block in `main.tf` at line 1 violated this constraint. Terraform detects the conflict and aborts initialization before downloading any provider plugins.

**Resolution:**

`main.tf` was overwritten a third time to contain only the `aws_sns_topic` resource block, removing the provider block entirely. `terraform init` then completed successfully.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
EOF
```

---

### Error 2: AWS CLI Connection Failure Using localhost Endpoint

**What happened:**

After `terraform apply` completed, the AWS CLI was invoked with `--endpoint-url=http://localhost:4566` to verify the topic in LocalStack:

```bash
aws --endpoint-url=http://localhost:4566 sns list-topics
```

```
Could not connect to the endpoint URL: "http://localhost:4566/"
```

**Root cause:**

The LocalStack container is reachable within this Docker network environment via the hostname `aws`, which is the container's network alias. The loopback address `localhost` does not resolve to the LocalStack container from inside the `iac-server` container. The correct hostname was already documented in `provider.tf` as `http://aws:4566` for all service endpoints, but was not applied when constructing the CLI verification command.

**Resolution:**

The endpoint URL was corrected to use the `aws` hostname:

```bash
aws --endpoint-url=http://aws:4566 sns list-topics
```

This returned the expected JSON response confirming the topic ARN.

---

## Best Practices Applied

* **Reviewed pre-existing configuration before writing new files.** Reading `provider.tf` before creating `main.tf` established a clear picture of what was already defined in the directory, informing the correct scope of the new file.

* **Separated provider configuration from resource definitions.** Keeping provider and backend configuration in `provider.tf` and resource definitions in `main.tf` follows the Terraform community file layout convention, prevents duplicate provider conflicts, and makes the workspace easier to maintain and hand off.

* **Ran terraform validate before terraform plan.** Configuration validation was performed as a discrete step before planning, catching structural errors at the earliest stage without requiring a network call to the provider endpoint.

* **Reviewed the execution plan before applying.** `terraform plan` was run before `terraform apply` to confirm the exact resource that would be created and to verify that no unintended modifications or deletions were included.

* **Verified Terraform state post-apply.** `terraform state list` was used immediately after apply to confirm the resource was registered in state, not just reported as created in the apply output.

* **Performed out-of-band CLI verification.** The AWS CLI was used independently of Terraform to query the LocalStack SNS endpoint directly, providing confirmation of resource existence outside of Terraform's internal state tracking.

* **Used `-auto-approve` only in a controlled non-production context.** The flag was appropriate here given the known, minimal scope of the plan. In production workflows, omitting `-auto-approve` is the standard to require explicit human confirmation of the plan before execution.

---

## Lessons Learned

* **Always inspect every `.tf` file in the working directory before writing new configuration.** In shared or pre-configured Terraform directories, provider blocks often already exist. Writing a duplicate `provider` block in a new file is one of the most common causes of `terraform init` failure, and while the error message is accurate, it does not immediately make clear which file should be corrected.

* **The LocalStack hostname is environment-specific, not universal.** In containerized setups, `localhost` frequently does not resolve to the LocalStack service. The correct hostname is determined by the container's network alias, which in this environment is `aws`. The existing `provider.tf` contained the correct hostname and should have been the reference for all CLI endpoint flags from the start.

* **Read the pre-existing provider configuration for endpoint details before writing CLI commands.** The `provider.tf` file explicitly mapped all service endpoints to `http://aws:4566`. Using that file as the source of truth for the CLI endpoint flag would have avoided the localhost connection failure entirely.

* **The `terraform init` duplicate provider error is reported twice but represents a single conflict.** Terraform surfaces the error once for each location where the duplicate is detected. Both messages point to the same root cause and require a single fix: removing the extra provider block from `main.tf`.

* **`terraform plan` output is reproduced in full at the start of `terraform apply`.** When using `-auto-approve`, Terraform re-displays the complete plan before executing it, allowing post-apply review of the exact actions that were taken even when no interactive confirmation step occurred.





<img width="1041" height="471" alt="image" src="https://github.com/user-attachments/assets/6951556f-b1ec-40c4-822f-6bba57716d1b" />
<img width="1029" height="691" alt="image" src="https://github.com/user-attachments/assets/fdeb825c-f1a8-45d9-a0a8-fa2d59a9dfac" />
<img width="1043" height="496" alt="image" src="https://github.com/user-attachments/assets/d606548a-2173-47ef-aa69-7e0aae29e1ab" />
<img width="1046" height="708" alt="image" src="https://github.com/user-attachments/assets/15daf16d-3476-4010-a86e-5cfafe489473" />
<img width="1051" height="303" alt="image" src="https://github.com/user-attachments/assets/085f14d0-418e-4b3c-9bf8-b0f5fb30f581" />
<img width="1050" height="301" alt="image" src="https://github.com/user-attachments/assets/af22cc51-b8b7-416e-b82f-6b51f68aa0a7" />
<img width="1048" height="597" alt="image" src="https://github.com/user-attachments/assets/d36d273d-6ef5-4941-ad22-0d82d13e512f" />
<img width="1050" height="676" alt="image" src="https://github.com/user-attachments/assets/ffdee858-d172-4e8c-8a6c-62d2f0579069" />
<img width="1040" height="661" alt="image" src="https://github.com/user-attachments/assets/d75dfb74-5c01-4756-ad1f-1f6db8349f72" />
<img width="1048" height="627" alt="image" src="https://github.com/user-attachments/assets/4f3248c2-4bf1-468d-8b28-4a86becf93e5" />
<img width="1044" height="620" alt="image" src="https://github.com/user-attachments/assets/871da1f2-88f1-495a-a2e1-08d256021035" />
<img width="1045" height="695" alt="image" src="https://github.com/user-attachments/assets/3f46e6a7-1a2a-4cac-a412-e1f67b4ca6f9" />
<img width="1047" height="424" alt="image" src="https://github.com/user-attachments/assets/75fe1320-6e62-4d40-9b96-ada267bd0f71" />
