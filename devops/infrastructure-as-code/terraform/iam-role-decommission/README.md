# Terraform IAM Role Deletion via Targeted Destroy

> **Domain:** Infrastructure as Code | AWS IAM | Terraform State Management
> **Environment:** LocalStack (AWS simulation via `http://aws:4566`)
> **Toolchain:** Terraform v1.x | AWS CLI | LocalStack

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Working Directory and Existing Files](#step-1-verify-working-directory-and-existing-files)
  - [Step 2: Inspect the Terraform Configuration](#step-2-inspect-the-terraform-configuration)
  - [Step 3: Confirm Current State](#step-3-confirm-current-state)
  - [Step 4: Run a Targeted Destroy Plan](#step-4-run-a-targeted-destroy-plan)
  - [Step 5: Execute the Targeted Destroy](#step-5-execute-the-targeted-destroy)
  - [Step 6: Confirm State Is Empty](#step-6-confirm-state-is-empty)
  - [Step 7: Verify Deletion Against the AWS Endpoint](#step-7-verify-deletion-against-the-aws-endpoint)
  - [Step 8: Confirm Provisioning Code Is Retained](#step-8-confirm-provisioning-code-is-retained)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Errors and Resolutions](#errors-and-resolutions)
- [References](#references)

---

## Overview

This project documents the controlled removal of an AWS IAM role named `iamrole_james` using Terraform's targeted destroy workflow. The operation was performed as part of a cleanup effort within the Nautilus DevOps team's AWS environment, where several resources had been provisioned for one-time use during a migration process.

The core constraint was intentional: the Terraform provisioning code must be preserved after deletion so the role can be re-provisioned at any future point without rewriting configuration from scratch. This distinguishes a **targeted destroy** from a full `terraform destroy`, which would have affected all managed resources.

---

## Problem Statement

The Nautilus DevOps team completed a migration process that involved provisioning several AWS resources intended for one-time use only. As part of post-migration hygiene, the IAM role `iamrole_james` was identified for removal. The requirements were:

* Delete the IAM role `iamrole_james` from the AWS environment (simulated via LocalStack).
* Use Terraform as the deletion mechanism to maintain state integrity.
* Retain the existing `main.tf` provisioning code so the role can be recreated on demand.

---

## Solution Architecture

The solution leverages Terraform's `-target` flag to perform a scoped destroy operation against a single named resource (`aws_iam_role.role`) without affecting any other infrastructure managed under the same state file. After destruction, verification is performed using the AWS CLI directly against the LocalStack endpoint to confirm the resource no longer exists at the API level.

```
Terraform State (pre-destroy)
+---------------------------+
|  aws_iam_role.role        |  <-- targeted for deletion
|  name: iamrole_james      |
+---------------------------+
          |
          v  terraform destroy -target=aws_iam_role.role
+---------------------------+
|  Terraform State (empty)  |
+---------------------------+
          |
          v  aws iam get-role (verification)
  NoSuchEntity: Role iamrole_james not found
```

The provisioning configuration (`main.tf`) remains untouched on disk, ready for future `terraform apply` operations.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | v1.x installed and initialized |
| AWS CLI | Configured with LocalStack endpoint access |
| LocalStack | Running and accessible at `http://aws:4566` |
| Working directory | `/home/bob/terraform` |
| Existing state | `aws_iam_role.role` tracked in `terraform.tfstate` |

---

## Repository Structure

```
/home/bob/terraform/
+-- main.tf                  # IAM role resource definition (retained post-deletion)
+-- provider.tf              # AWS provider config pointing to LocalStack endpoints
+-- .terraform/              # Provider plugins and module cache
+-- .terraform.lock.hcl      # Dependency lock file
+-- terraform.tfstate         # Active Terraform state file
+-- terraform.tfstate.backup  # Auto-generated backup created after destroy
+-- README.MD                # Original task description
```

---

## Implementation Guide

### Step 1: Verify Working Directory and Existing Files

Navigate to the Terraform working directory and confirm the expected files are present before executing any operations.

```bash
ls -la
```

**Expected output:**

```
total 40
drwxr-xr-x 1 bob bob 4096 May  6 21:41 .
drwxr-x--- 1 bob bob 4096 May  6 21:41 ..
drwxr-xr-x 3 bob bob 4096 May  6 21:41 .terraform
-rw-r--r-- 1 bob bob 1406 May  6 21:41 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  353 May  6 21:41 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-rw-r-- 1 bob bob 1396 May  6 21:41 terraform.tfstate
```

> Screenshot: `01-working-directory-ls-la.png`

---

### Step 2: Inspect the Terraform Configuration

Review the `main.tf` to confirm the resource being targeted. This also serves as documentation that the configuration will be preserved post-deletion.

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_iam_role" "role" {
  name = "iamrole_james"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = {
    Name = "iamrole_james"
  }
}
```

The role is defined with an EC2 trust policy, enabling EC2 instances to assume this role via `sts:AssumeRole`.

> Screenshot: `02-cat-main-tf.png`

---

### Step 3: Confirm Current State

Before executing any destructive operation, list the resources tracked in the Terraform state file to confirm the target resource is present.

```bash
terraform state list
```

**Output:**

```
aws_iam_role.role
```

This confirms that `aws_iam_role.role` is the only tracked resource and is the intended deletion target.

> Screenshot: `03-terraform-state-list.png`

---

### Step 4: Run a Targeted Destroy Plan

Execute a dry-run destroy plan scoped to the specific resource. This produces a preview of the actions Terraform will take without making any changes to infrastructure.

```bash
terraform plan -destroy -target=aws_iam_role.role
```

**Key output excerpt:**

```
  # aws_iam_role.role will be destroyed
  - resource "aws_iam_role" "role" {
      - arn    = "arn:aws:iam::000000000000:role/iamrole_james" -> null
      - name   = "iamrole_james" -> null
      ...
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

Terraform confirms exactly one resource will be destroyed and zero resources will be created or modified. The `-destroy` flag combined with `-target` produces a safe, auditable preview before committing.

> Screenshot: `04-terraform-plan-destroy-target.png`

---

### Step 5: Execute the Targeted Destroy

With the plan validated, execute the actual destroy operation targeting only `aws_iam_role.role`.

```bash
terraform destroy -target=aws_iam_role.role
```

When prompted for confirmation, type `yes` and press Enter.

```
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

**Output upon completion:**

```
aws_iam_role.role: Destroying... [id=iamrole_james]
aws_iam_role.role: Destruction complete after 0s

Destroy complete! Resources: 1 destroyed.
```

> Screenshot: `05-terraform-destroy-target-confirm-yes.png`

> Screenshot: `06-terraform-destroy-complete.png`

---

### Step 6: Confirm State Is Empty

After the destroy completes, re-run `terraform state list` to confirm the state file no longer tracks any resources.

```bash
terraform state list
```

**Output:**

```
(empty)
```

No output is returned, confirming the state file has been fully cleared of tracked resources.

> Screenshot: `07-terraform-state-list-empty.png`

---

### Step 7: Verify Deletion Against the AWS Endpoint

Use the AWS CLI to query the IAM service directly via the LocalStack endpoint and confirm the role no longer exists at the API level.

```bash
aws --endpoint-url=http://aws:4566 iam get-role --role-name iamrole_james
```

**Output:**

```
An error occurred (NoSuchEntity) when calling the GetRole operation: Role iamrole_james not found
```

The `NoSuchEntity` response from the IAM API is the definitive confirmation that the role has been fully deleted from the simulated AWS environment.

> Screenshot: `08-aws-cli-get-role-no-such-entity.png`

---

### Step 8: Confirm Provisioning Code Is Retained

Verify that `main.tf` still exists and contains the full role definition, ensuring future reprovisioning is possible without any configuration recovery effort.

```bash
ls -la
cat main.tf
```

**Expected ls output (post-destroy):**

```
total 44
drwxr-xr-x 1 bob bob 4096 May  6 21:49 .
drwxr-x--- 1 bob bob 4096 May  6 21:41 ..
drwxr-xr-x 3 bob bob 4096 May  6 21:41 .terraform
-rw-r--r-- 1 bob bob 1406 May  6 21:41 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  353 May  6 21:41 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-r--r-- 1 bob bob  181 May  6 21:49 terraform.tfstate
-rw-r--r-- 1 bob bob 1396 May  6 21:49 terraform.tfstate.backup
```

Notable changes from the initial state:

* `terraform.tfstate` is now smaller (181 bytes vs 1396 bytes), reflecting an empty resource list.
* `terraform.tfstate.backup` has been automatically created by Terraform, preserving the pre-destroy state snapshot.
* `main.tf` is unchanged at 353 bytes, confirming the provisioning code is intact.

> Screenshot: `09-post-destroy-ls-la-cat-main-tf.png`

---

## Best Practices Applied

**Dry-run before destroy**
A `terraform plan -destroy -target` was executed before the actual `terraform destroy`. This is a non-negotiable step in production workflows, as it surfaces the exact resource actions Terraform will take and allows for peer review or audit before any irreversible changes are committed.

**Targeted destroy over full destroy**
Using `-target=aws_iam_role.role` scopes the operation to a single resource instead of tearing down all state-managed infrastructure. This minimises blast radius and is the correct approach when only a subset of resources needs to be removed.

**Provisioning code preservation**
The `main.tf` was intentionally left in place after deletion. Keeping the Terraform definition on disk means the role can be recreated with a single `terraform apply` without any manual reconstruction of configuration. This is a standard pattern in environments where resources have lifecycle-driven on/off requirements.

**State verification before and after**
Running `terraform state list` both before and after the destroy operation provides a clear audit trail: one resource tracked, then zero. This is a simple but critical step in confirming that Terraform's state file accurately reflects the infrastructure reality.

**API-level verification**
Using the AWS CLI `iam get-role` command directly against the LocalStack endpoint provides independent confirmation beyond Terraform's own reporting. This closes the loop between Terraform state and actual provider-side resource existence.

**State backup awareness**
Terraform automatically creates `terraform.tfstate.backup` on destroy. This file contains the pre-destroy state and serves as a manual recovery point if the deletion was performed in error. Its presence was confirmed in the post-destroy directory listing.

---

## Lessons Learned

**Targeted destroy warnings are expected, not errors**
Terraform emits `Warning: Resource targeting is in effect` and `Warning: Applied changes may be incomplete` when using `-target`. These are informational warnings designed to remind engineers that `-target` is not intended for routine use. They do not indicate a problem with the execution and should not cause concern when the targeting is deliberate and understood.

**State file size is a useful post-operation signal**
The `terraform.tfstate` file dropped from 1396 bytes to 181 bytes after the destroy. Monitoring state file size after destructive operations is a quick sanity check that the operation completed as intended, before running heavier verification commands.

**The `-target` flag is scoped to plan and apply, not the state file globally**
When `-target` is used with `destroy`, Terraform removes the targeted resource from state and destroys it from the provider. Resources not in the target remain in state untouched. This is important to understand in environments with multiple resources managed by the same state file.

**Provisioning code is not coupled to resource existence**
A common misconception is that the Terraform code must be removed when a resource is deleted. In reality, `terraform destroy` removes the resource from the provider and from state, but leaves the `.tf` files completely untouched. The code is a blueprint, not a live reference. Retaining it is always the correct approach unless the resource is being permanently decommissioned with no future provisioning intent.

**LocalStack endpoint verification closes the control loop**
Relying solely on `terraform state list` to confirm deletion is insufficient. The state file reflects Terraform's view of the world, not the provider's. Querying the AWS CLI directly via the LocalStack endpoint (`--endpoint-url=http://aws:4566`) verifies that the deletion propagated correctly to the simulated AWS API layer, providing a higher confidence assertion than state inspection alone.

---

## Errors and Resolutions

**Warning: Resource targeting is in effect**

* **Encountered during:** `terraform plan -destroy -target` and `terraform destroy -target`
* **Symptom:** Terraform printed warning messages about targeting being in effect and that applied changes may be incomplete.
* **Root cause:** These are standard Terraform informational warnings emitted whenever `-target` is used. They are not errors and do not indicate a failed operation.
* **Resolution:** No action required. The warnings are expected behavior and serve as a reminder that `-target` is a scoped operation, not a full reconciliation. They can be safely acknowledged and ignored when the targeting is intentional.

---

## References

* [Terraform Resource Targeting Documentation](https://developer.hashicorp.com/terraform/cli/commands/plan#target-address)
* [AWS IAM Role Terraform Resource](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_role)
* [LocalStack AWS Simulation Documentation](https://docs.localstack.cloud/overview/)
* [AWS CLI IAM get-role Command Reference](https://docs.aws.amazon.com/cli/latest/reference/iam/get-role.html)
* [Terraform State Management](https://developer.hashicorp.com/terraform/language/state)


<img width="1036" height="691" alt="image" src="https://github.com/user-attachments/assets/0229b82e-d13e-4833-a351-fb8f5286dfa7" />
<img width="1045" height="704" alt="image" src="https://github.com/user-attachments/assets/d32b10b3-4647-4c27-8632-e8944f0ab2d2" />
<img width="1070" height="817" alt="image" src="https://github.com/user-attachments/assets/5bb3e0b8-b3b1-468d-b1b9-f5b7afb5028d" />
<img width="1077" height="636" alt="image" src="https://github.com/user-attachments/assets/d128cba7-5cfd-4c66-aa2d-b77338adfd69" />
<img width="1076" height="809" alt="image" src="https://github.com/user-attachments/assets/67b36f4f-22db-42b0-82c0-212641ebf579" />
<img width="1077" height="807" alt="image" src="https://github.com/user-attachments/assets/3f4833b5-0935-4af0-8980-2031e2cec2c5" />
<img width="1072" height="457" alt="image" src="https://github.com/user-attachments/assets/302a6634-12af-4d9d-8fcb-bcaf9c18d3e7" />
<img width="1077" height="353" alt="image" src="https://github.com/user-attachments/assets/61051eb5-848b-424b-816a-742a3ecf932a" />
<img width="1076" height="403" alt="image" src="https://github.com/user-attachments/assets/112449a9-7424-4025-9693-d9dee6b28a04" />
<img width="1070" height="766" alt="image" src="https://github.com/user-attachments/assets/6ce3e4ff-f886-41d5-9021-e2a69fdce680" />
