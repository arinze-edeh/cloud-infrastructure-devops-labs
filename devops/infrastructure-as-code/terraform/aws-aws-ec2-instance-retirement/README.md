# Terraform Targeted EC2 Instance Destruction with State Verification

> Safely destroying a tagged EC2 instance in AWS using Terraform's targeted destroy workflow while preserving the original provisioning configuration for future reuse.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture and Context](#architecture-and-context)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify the Working Directory](#step-1-verify-the-working-directory)
  - [Step 2: Review the Provisioning Configuration](#step-2-review-the-provisioning-configuration)
  - [Step 3: List Resources in Terraform State](#step-3-list-resources-in-terraform-state)
  - [Step 4: Preview the Destroy Plan](#step-4-preview-the-destroy-plan)
  - [Step 5: Execute the Targeted Destroy](#step-5-execute-the-targeted-destroy)
  - [Step 6: Verify Instance Termination via AWS CLI](#step-6-verify-instance-termination-via-aws-cli)
  - [Step 7: Confirm Provisioning Code Is Preserved](#step-7-confirm-provisioning-code-is-preserved)
- [Best Practices](#best-practices)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

During infrastructure migration, test resources are often provisioned to validate configurations and workflows. Once those resources are no longer actively needed, they must be cleanly decommissioned to avoid unnecessary cost and drift. This implementation documents the process of destroying a specific EC2 instance (`nautilus-ec2`) in the `us-east-1` region using a Terraform targeted destroy operation, while retaining the original `main.tf` configuration intact for future reprovisioning.

---

## Problem Statement

A migration process created several AWS resources, including an EC2 instance tagged `nautilus-ec2`. This instance is no longer required and must be terminated. Two constraints drive the approach:

1. The destruction must be handled through Terraform to maintain state consistency, not the AWS Console or CLI.
2. The provisioning configuration (`main.tf`) must be preserved post-destroy, as the instance may need to be reprovisioned at a later date.

---

## Architecture and Context

| Attribute | Value |
|---|---|
| Resource Type | EC2 Instance |
| Resource Name (Terraform) | `aws_instance.ec2` |
| Tag: Name | `nautilus-ec2` |
| AMI | `ami-0c101f26f147fa7fd` |
| Instance Type | `t2.micro` |
| Region | `us-east-1` |
| Availability Zone | `us-east-1a` |
| Security Group | `sg-ca6d8df85e10fc197` |
| Terraform Working Directory | `/home/bob/terraform` |
| State Backend | Local (`terraform.tfstate`) |

The Terraform workspace contained a pre-initialized configuration with the provider already configured and the instance tracked in state. The destroy operation targeted only `aws_instance.ec2`, leaving all other state entries and configuration files untouched.

---

## Prerequisites

- Terraform installed and initialized in the working directory
- AWS credentials configured with permissions to terminate EC2 instances and describe instance state
- AWS CLI installed for post-destroy verification
- The target resource (`aws_instance.ec2`) present in `terraform.tfstate`

---

## Implementation Guide

### Step 1: Verify the Working Directory

Navigate to the Terraform working directory and confirm all expected configuration files are present before proceeding.

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
total 44
drwxr-xr-x 1 bob bob 4096 May  2 07:42 .
drwxr-x--- 1 bob bob 4096 May  2 07:42 ..
drwxr-xr-x 3 bob bob 4096 May  2 07:42 .terraform
-rw-r--r-- 1 bob bob 1406 May  2 07:42 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  231 May  2 07:42 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-r--r-- 1 bob bob 4099 May  2 07:42 terraform.tfstate
```

The presence of `.terraform/`, `.terraform.lock.hcl`, `main.tf`, `provider.tf`, and `terraform.tfstate` confirms the workspace is initialized and the instance is currently tracked in state.

*Screenshot: Terminal output of `ls -la` showing all workspace files*

---

### Step 2: Review the Provisioning Configuration

Inspect `main.tf` to confirm the resource definition before performing any destructive operation.

```bash
cat main.tf
```

**Output:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  vpc_security_group_ids = [
    "sg-ca6d8df85e10fc197"
  ]

  tags = {
    Name = "nautilus-ec2"
  }
}
```

This file defines the EC2 instance using a fixed AMI, `t2.micro` instance type, and a named security group. The `Name` tag `nautilus-ec2` matches the resource targeted for destruction. This configuration is retained as-is throughout the process.

*Screenshot: Terminal output of `cat main.tf` showing the resource block*

---

### Step 3: List Resources in Terraform State

Confirm that `aws_instance.ec2` is tracked in the current Terraform state before proceeding.

```bash
terraform state list
```

**Output:**

```
aws_instance.ec2
```

Only one resource is present in state. This confirms the targeted destroy will operate on the correct and only managed resource.

*Screenshot: Terminal output of `terraform state list`*

---

### Step 4: Preview the Destroy Plan

Run a targeted plan with the `-destroy` flag to preview exactly what Terraform will remove. This is a non-destructive operation and must be reviewed before applying.

```bash
terraform plan -destroy -target=aws_instance.ec2
```

**Output (key excerpt):**

```
aws_instance.ec2: Refreshing state... [id=i-a9d81f2edbac50866]

Terraform used the selected providers to generate the following execution plan.

  # aws_instance.ec2 will be destroyed
  - resource "aws_instance" "ec2" {
      - ami                                  = "ami-0c101f26f147fa7fd" -> null
      - arn                                  = "arn:aws:ec2:us-east-1::instance/i-a9d81f2edbac50866" -> null
      - instance_type                        = "t2.micro" -> null
      - instance_state                       = "running" -> null
      - public_ip                            = "54.214.64.205" -> null
      - private_ip                           = "10.85.160.151" -> null
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

The plan confirms:

- One resource (`aws_instance.ec2`) will be destroyed
- No resources will be added or modified
- The instance is in a `running` state at the time of planning
- The instance ID is `i-a9d81f2edbac50866`

Terraform issued a warning that resource targeting is in effect. This is expected behavior for `-target` operations and does not indicate an error.

*Screenshot: Full terminal output of `terraform plan -destroy -target=aws_instance.ec2`*

---

### Step 5: Execute the Targeted Destroy

Apply the targeted destroy to remove the EC2 instance from AWS and from Terraform state.

```bash
terraform destroy -target=aws_instance.ec2
```

Terraform displayed the full destroy plan again and prompted for confirmation:

```
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

Entered `yes` to confirm.

**Output:**

```
aws_instance.ec2: Destroying... [id=i-a9d81f2edbac50866]
aws_instance.ec2: Still destroying... [id=i-a9d81f2edbac50866, 10s elapsed]
aws_instance.ec2: Destruction complete after 11s

Destroy complete! Resources: 1 destroyed.
```

The instance was fully destroyed in 11 seconds. Terraform removed the resource from state automatically upon successful destruction.

*Screenshot: Terminal output showing the confirmation prompt and destruction progress*

*Screenshot: Terminal output showing `Destroy complete! Resources: 1 destroyed.`*

---

### Step 6: Verify Instance Termination via AWS CLI

Confirm that the EC2 instance reached `terminated` state in AWS by querying directly through the AWS CLI using the resource tag.

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name}" \
  --output table
```

**Output:**

```
---------------------------------------
|          DescribeInstances          |
+----------------------+--------------+
|          ID          |    State     |
+----------------------+--------------+
|  i-a9d81f2edbac50866 |  terminated  |
+----------------------+--------------+
```

The instance `i-a9d81f2edbac50866` is confirmed in `terminated` state. This matches the instance ID tracked in Terraform state and validates that the destroy operation completed successfully in AWS.

*Screenshot: AWS CLI table output showing `terminated` state for `nautilus-ec2`*

---

### Step 7: Confirm Provisioning Code Is Preserved

After destruction, verify that `main.tf` still contains the original resource definition and was not modified during the destroy process.

```bash
cat main.tf
```

**Output:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  vpc_security_group_ids = [
    "sg-ca6d8df85e10fc197"
  ]

  tags = {
    Name = "nautilus-ec2"
  }
}
```

The provisioning configuration is fully intact and unchanged. Running `terraform apply` in the future against this configuration will reprovision an identical EC2 instance.

*Screenshot: Terminal output of `cat main.tf` post-destroy, confirming the resource block is unchanged*

---

## Best Practices

**Always preview before destroying.** Running `terraform plan -destroy -target=<resource>` before `terraform destroy` gives a precise view of what will be removed, including all attributes and nested blocks. Never skip this step in a production or shared environment.

**Use `-target` only for exceptional cases.** Terraform itself warns that `-target` is not for routine use. It is appropriate when decommissioning a single resource during a migration cleanup while other infrastructure remains active and managed.

**Verify state before and after.** Checking `terraform state list` before the destroy confirms the resource is tracked. After destruction, Terraform automatically removes the resource from state, which can be validated with another `terraform state list` call showing an empty or reduced list.

**Do not delete `main.tf` after destruction.** Retaining the provisioning code ensures the instance can be reprovisioned without reconstructing the configuration from memory or documentation. The separation between Terraform state and configuration code is a core design principle.

**Use tag-based CLI verification.** Querying AWS with `--filters "Name=tag:Name,Values=<name>"` is more reliable than querying by instance ID alone, as it validates both the tag and the terminal state match the expected resource.

**Capture instance metadata from the plan output.** The destroy plan surfaces runtime attributes including instance ID, public IP, private IP, subnet, and AMI. This output should be captured before confirming destruction in case the information is needed for audit trails or incident review.

---

## Errors and Resolutions

No errors were encountered during this implementation. The following scenarios represent common issues in similar targeted destroy workflows and their resolutions:

**Scenario: `Error: No such resource instance in the state`**

This occurs if the resource address passed to `-target` does not match the resource in state. Resolution: run `terraform state list` to retrieve the exact resource address and use it verbatim in the `-target` flag.

**Scenario: `Error refreshing state: NoCredentialProviders`**

This occurs if AWS credentials are not configured or have expired. Resolution: verify credentials using `aws sts get-caller-identity` and reconfigure via environment variables or `~/.aws/credentials` before retrying.

**Scenario: Instance shows `shutting-down` instead of `terminated` in the AWS CLI query**

This is a transient state. EC2 instances pass through `shutting-down` before reaching `terminated`. Wait 30 to 60 seconds and re-run the describe-instances query.

---

## Lessons Learned

**The `-out` flag for plan files is valuable in team environments.** Terraform warned that without `-out`, it cannot guarantee the apply matches the plan exactly. In shared or regulated environments, always save the plan with `-out=destroy.tfplan` and apply from the saved plan using `terraform apply destroy.tfplan`.

**`terraform destroy -target` removes the resource from state automatically.** Unlike `terraform state rm`, which removes a resource from state without destroying the actual infrastructure, `terraform destroy` performs both the AWS API call and the state cleanup in a single atomic operation. This is the correct approach when the intent is full decommissioning.

**AWS CLI verification is a necessary final gate.** Terraform reporting success does not guarantee the AWS resource reached `terminated` state before the CLI check. The 11-second destruction time in this implementation included the AWS API call completing. Using `describe-instances` with the tag filter and `--output table` produces a clean, unambiguous confirmation that can be included in change records or runbooks.

**Preserving configuration after destruction is a reproducibility discipline.** The decision to keep `main.tf` intact after destroying the instance reflects a key IaC principle: the configuration describes desired state, while the state file tracks actual state. Destroying infrastructure does not invalidate the configuration, and teams should treat `main.tf` files as living documents independent of deployment lifecycle.

**Targeted operations create incomplete apply context.** Terraform's warning that "applied changes may be incomplete" after a targeted destroy is accurate. Any subsequent `terraform plan` should be run without `-target` to check for pending drift or mismatches between the configuration and remaining state.









<img width="1025" height="513" alt="image" src="https://github.com/user-attachments/assets/7e890f5b-61e6-41ef-9281-674638dfaeca" />
<img width="1027" height="629" alt="image" src="https://github.com/user-attachments/assets/93e9cb94-f243-47f6-8b96-8dc8bde3003a" />
<img width="1033" height="707" alt="image" src="https://github.com/user-attachments/assets/e2f437c1-f805-4363-8dd9-668bce93aede" />
<img width="1031" height="717" alt="image" src="https://github.com/user-attachments/assets/613d25e0-f151-4650-b51a-1ac1bd825a35" />
<img width="1076" height="818" alt="image" src="https://github.com/user-attachments/assets/f4e1d96b-5df7-4ce3-a04f-ea5cae4ef4d8" />
<img width="1071" height="818" alt="image" src="https://github.com/user-attachments/assets/3f7a4f5b-0c1c-416c-a706-6af9f0264387" />
<img width="1075" height="796" alt="image" src="https://github.com/user-attachments/assets/a11a64d2-ed14-483c-b6d0-bc9365e4c58f" />
<img width="1074" height="814" alt="image" src="https://github.com/user-attachments/assets/281cbe90-20e1-4ab2-b79d-8b102e21d518" />
<img width="1048" height="584" alt="image" src="https://github.com/user-attachments/assets/6f15374a-f50e-4a7e-ad92-cb20c67a7136" />
<img width="1051" height="546" alt="image" src="https://github.com/user-attachments/assets/07a752f4-b9ea-4c44-98ef-aa9db67f4a77" />








