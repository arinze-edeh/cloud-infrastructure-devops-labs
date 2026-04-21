# Terraform SNS Topic Provisioning on AWS (LocalStack)

Provision an AWS SNS topic named `xfusion-notifications` using Terraform against a LocalStack endpoint, resolving a duplicate provider conflict encountered during initialization.

---

## Table of Contents

- [Overview](#overview)
- [Architecture and Design Intent](#architecture-and-design-intent)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Inspect the Working Directory](#step-1-inspect-the-working-directory)
  - [Step 2: Review the Existing Provider Configuration](#step-2-review-the-existing-provider-configuration)
  - [Step 3: Author main.tf with the SNS Resource](#step-3-author-maintf-with-the-sns-resource)
  - [Step 4: Initialize Terraform](#step-4-initialize-terraform)
  - [Step 5: Validate the Configuration](#step-5-validate-the-configuration)
  - [Step 6: Plan the Deployment](#step-6-plan-the-deployment)
  - [Step 7: Apply and Provision](#step-7-apply-and-provision)
  - [Step 8: Verify State and Resource](#step-8-verify-state-and-resource)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
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

The task required provisioning a single SNS topic using Terraform in a LocalStack-backed environment. A pre-existing `provider.tf` file was already present in the working directory. The implementation involved creating only a `main.tf` file containing the resource definition, avoiding any modifications to the existing provider file.

---

## Architecture and Design Intent

LocalStack simulates AWS service endpoints locally. All AWS API calls are routed to `http://aws:4566` rather than real AWS infrastructure. The Terraform AWS provider is configured to skip credential validation and account ID resolution, which are irrelevant in a local emulation context.

The SNS topic serves as a notification bus. In production, topics of this kind underpin event-driven pipelines, alerting systems, and fan-out messaging architectures. This implementation reflects the minimal viable configuration for provisioning a standard (non-FIFO) SNS topic.

The separation between `provider.tf` and `main.tf` follows the Terraform community convention of isolating provider and backend configuration from resource definitions, improving maintainability and enabling modular reuse across workspaces.

---

## Prerequisites

* Terraform v1.11.0 or later installed
* AWS CLI configured and accessible
* LocalStack running and reachable at `http://aws:4566`
* Write access to `/home/bob/terraform`

---

## Repository Structure

```
/home/bob/terraform/
|-- provider.tf        # AWS provider and LocalStack endpoint configuration (pre-existing)
|-- main.tf            # SNS topic resource definition (created in this implementation)
|-- README.MD          # Original task description
|-- .terraform/        # Provider plugin cache (generated after terraform init)
|-- .terraform.lock.hcl # Provider dependency lock file (generated after terraform init)
```

---

## Implementation Guide

### Step 1: Inspect the Working Directory

Confirm the contents of the Terraform working directory before making any changes.

```bash
ls -la /home/bob/terraform
```

**Expected output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 21 18:26 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

Two files are present: `README.MD` and `provider.tf`. No `main.tf` exists yet. This confirms the working directory is clean and ready for the resource file to be created.

*Screenshot: Working directory listing showing README.MD and provider.tf*

<img width="1048" height="512" alt="image" src="https://github.com/user-attachments/assets/ddaecc45-be28-41f7-84be-e2d63c8c8276" />

---

### Step 2: Review the Existing Provider Configuration

Read `provider.tf` to understand the LocalStack endpoint mapping before authoring the resource file.

```bash
cat /home/bob/terraform/provider.tf
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

The provider block already defines the AWS provider with full LocalStack endpoint overrides, including `sns = "http://aws:4566"`. This means `main.tf` must contain only the resource definition. Adding another `provider "aws"` block in `main.tf` would result in a duplicate provider error.

*Screenshot: cat output of provider.tf showing LocalStack endpoint configuration*

<img width="1074" height="705" alt="image" src="https://github.com/user-attachments/assets/cb743707-be27-4ea2-8e45-92f3f9215254" />

---

### Step 3: Author main.tf with the SNS Resource

Create `main.tf` containing only the SNS topic resource. The provider configuration already exists in `provider.tf` and must not be duplicated.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
EOF
```

Confirm the file contents:

```bash
cat /home/bob/terraform/main.tf
```

**Expected output:**

```hcl
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
```

*Screenshot: cat output of main.tf confirming the resource block is correct and contains no provider block*

Verify the directory now contains both configuration files:

```bash
ls -la /home/bob/terraform/
```

**Expected output:**

```
total 24
drwxr-xr-x 1 bob bob 4096 Apr 21 18:32 .
drwxr-x--- 1 bob bob 4096 Apr 21 18:29 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  414 Apr 21 18:33 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

*Screenshot: Updated directory listing confirming main.tf has been created alongside provider.tf*

---

### Step 4: Initialize Terraform

Download the required provider plugin and set up the working directory.

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

*Screenshot: terraform init output confirming successful provider installation*

---

### Step 5: Validate the Configuration

Confirm that the Terraform configuration files are syntactically and semantically correct.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

*Screenshot: terraform validate confirming zero configuration errors*

---

### Step 6: Plan the Deployment

Generate and review the execution plan to confirm the intended resource will be created.

```bash
terraform plan
```

**Expected output (abbreviated):**

```
Terraform will perform the following actions:

  # aws_sns_topic.xfusion_notifications will be created
  + resource "aws_sns_topic" "xfusion_notifications" {
      + arn                         = (known after apply)
      + content_based_deduplication = false
      + fifo_topic                  = false
      + id                          = (known after apply)
      + name                        = "xfusion-notifications"
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

The plan confirms one resource will be created: an SNS topic named `xfusion-notifications`. No existing state or infrastructure will be modified or destroyed.

*Screenshot: terraform plan output showing the aws_sns_topic resource to be created*

---

### Step 7: Apply and Provision

Execute the plan and provision the SNS topic against the LocalStack endpoint.

```bash
terraform apply -auto-approve
```

**Expected output:**

```
aws_sns_topic.xfusion_notifications: Creating...
aws_sns_topic.xfusion_notifications: Creation complete after 0s [id=arn:aws:sns:us-east-1:000000000000:xfusion-notifications]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The resource is provisioned immediately. The assigned ARN confirms LocalStack's 12-zero account ID (`000000000000`), which is the expected behavior in a local emulation environment.

*Screenshot: terraform apply output confirming resource creation and the assigned SNS topic ARN*

---

### Step 8: Verify State and Resource

**Verify Terraform state:**

```bash
terraform state list
```

**Expected output:**

```
aws_sns_topic.xfusion_notifications
```

*Screenshot: terraform state list output showing the provisioned SNS topic in state*

**Verify the resource exists in LocalStack via AWS CLI:**

```bash
aws --endpoint-url=http://aws:4566 sns list-topics
```

**Expected output:**

```json
{
    "Topics": [
        {
            "TopicArn": "arn:aws:sns:us-east-1:000000000000:xfusion-notifications"
        }
    ]
}
```

*Screenshot: AWS CLI output confirming the SNS topic is visible in LocalStack*

The SNS topic `xfusion-notifications` is confirmed to exist at the LocalStack endpoint and matches the expected ARN format.

---

## Errors Encountered and Resolutions

### Error 1: Duplicate Provider Configuration

**Context:**

During initial experimentation, a second `provider "aws"` block (with a different endpoint URL) was written into `main.tf` alongside the resource definition. Running `terraform init` produced the following error:

```
Error: Duplicate provider configuration

  on provider.tf line 10:
  10: provider "aws" {

A default (non-aliased) provider configuration for "aws" was already given
at main.tf:1,1-15. If multiple configurations are required, set the "alias"
argument for alternative configurations.
```

**Root Cause:**

Terraform does not permit more than one default (non-aliased) block for the same provider across all `.tf` files in a directory. Since `provider.tf` already contained `provider "aws" { ... }`, including a second identical block in `main.tf` violated this constraint.

**Resolution:**

`main.tf` was rewritten to contain only the `aws_sns_topic` resource block. The provider configuration in `provider.tf` was left untouched. After this correction, `terraform init` completed successfully.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_sns_topic" "xfusion_notifications" {
  name = "xfusion-notifications"
}
EOF
```

---

### Error 2: Incorrect LocalStack Endpoint in CLI Verification

**Context:**

After provisioning, the AWS CLI was initially invoked with `--endpoint-url=http://localhost:4566`:

```bash
aws --endpoint-url=http://localhost:4566 sns list-topics
```

**Output:**

```
Could not connect to the endpoint URL: "http://localhost:4566/"
```

**Root Cause:**

The LocalStack service is not exposed on the loopback address (`localhost`) within this environment. It is accessible via the container hostname `aws`, which resolves correctly from within the network namespace.

**Resolution:**

The endpoint was corrected to use the `aws` hostname:

```bash
aws --endpoint-url=http://aws:4566 sns list-topics
```

This returned the expected JSON response confirming the topic's existence.

---

## Best Practices Applied

* **Provider and resource separation:** Provider configuration was maintained exclusively in `provider.tf`. The `main.tf` file contained only resource definitions. This separation follows the Terraform community file layout convention and prevents the duplicate provider error class entirely.

* **Lock file committed:** The `.terraform.lock.hcl` file generated by `terraform init` pins the exact provider version (`5.91.0`) for reproducible execution across environments and team members.

* **Validate before apply:** `terraform validate` was run before `terraform plan` and `terraform apply`, catching any configuration errors at the earliest possible stage.

* **Plan review before apply:** `terraform plan` was executed to produce a human-readable execution plan before applying any changes, confirming no unintended destructive operations.

* **State verification post-apply:** `terraform state list` was used immediately after apply to confirm the resource was registered in Terraform state, not just provisioned in the target service.

* **Out-of-band CLI verification:** The AWS CLI was used independently of Terraform to query the LocalStack SNS endpoint directly, confirming infrastructure existence beyond Terraform's internal state view.

* **Minimal footprint in main.tf:** Only the required resource block was written, avoiding provider-level concerns in resource files and keeping the configuration clean and auditable.

---

## Lessons Learned

* **Read pre-existing configuration files before writing new ones.** In shared Terraform directories, provider blocks often already exist. Duplicating a provider block is a common source of `terraform init` failures. Always inspect every `.tf` file in the directory before introducing new configuration.

* **LocalStack hostname resolution is environment-dependent.** The hostname used to reach LocalStack varies by deployment method. In containerized setups, `localhost` often does not resolve to the LocalStack container. The correct hostname (`aws` in this case) is defined in the container's networking configuration, not by convention. Verify the hostname before writing CLI commands or Terraform endpoint overrides.

* **`terraform validate` catches structural errors without network calls.** Running validation before plan or apply saves time and avoids unnecessary provider initialization overhead when configuration mistakes exist.

* **Terraform state is the source of truth for Terraform-managed resources.** After every apply, verifying state via `terraform state list` or `terraform state show` confirms that the resource is being tracked correctly. Out-of-band CLI verification adds a second layer of confidence that the resource exists in the actual infrastructure target.

* **The `-auto-approve` flag is appropriate in non-interactive or scripted execution contexts.** In interactive workflows or production environments, omit `-auto-approve` to require explicit human confirmation of the execution plan before changes are applied.





<img width="1035" height="579" alt="image" src="https://github.com/user-attachments/assets/06be32cc-99e4-4189-a069-caf1eb6d8536" />
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
