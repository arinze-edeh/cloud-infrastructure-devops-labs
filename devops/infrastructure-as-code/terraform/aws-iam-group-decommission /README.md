# Terraform Targeted Destroy: IAM Group Deprovisioning with State Preservation

> Deprovisioning an AWS IAM group via a Terraform targeted destroy operation while preserving the provisioning configuration for future re-use, executed against a LocalStack-backed AWS environment.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Implementation Guide](#implementation-guide)
  - [Step 1: Verify Working Directory and Project Files](#step-1-verify-working-directory-and-project-files)
  - [Step 2: Confirm the Target Resource in Configuration](#step-2-confirm-the-target-resource-in-configuration)
  - [Step 3: Inspect Current Terraform State](#step-3-inspect-current-terraform-state)
  - [Step 4: Back Up the Configuration File](#step-4-back-up-the-configuration-file)
  - [Step 5: Preview the Destroy Operation](#step-5-preview-the-destroy-operation)
  - [Step 6: Execute the Targeted Destroy](#step-6-execute-the-targeted-destroy)
  - [Step 7: Confirm State is Empty](#step-7-confirm-state-is-empty)
  - [Step 8: Verify Deletion via AWS CLI](#step-8-verify-deletion-via-aws-cli)
  - [Step 9: Validate Configuration Integrity Post-Destroy](#step-9-validate-configuration-integrity-post-destroy)
  - [Step 10: Run terraform plan to Confirm Re-Provisioning Readiness](#step-10-run-terraform-plan-to-confirm-re-provisioning-readiness)
  - [Step 11: Clean Up the Backup File](#step-11-clean-up-the-backup-file)
  - [Step 12: Final Directory Inspection](#step-12-final-directory-inspection)
- [Best Practices](#best-practices)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)

---

## Overview

This project documents the targeted deprovisioning of an AWS IAM group (`iamgroup_rose`) using Terraform's `-target` flag. The operation removes the live resource from AWS and clears it from Terraform state, while intentionally retaining the `.tf` configuration file to enable future re-provisioning without rewriting infrastructure code.

This pattern is commonly applied during cloud environment cleanup cycles where resources were provisioned for one-time migrations, staging workflows, or temporary team structures and are no longer required in their current form.

---

## Problem Statement

The Nautilus DevOps team completed a migration phase that required the temporary creation of several AWS IAM groups. With the migration complete, these groups are no longer needed and must be removed to reduce IAM surface area, enforce least-privilege hygiene, and eliminate unused resources from the AWS account.

The specific requirements for this cleanup operation are:

* Remove the IAM group named `iamgroup_rose` from AWS
* Use Terraform as the authoritative tool for the destroy operation to maintain state consistency
* **Preserve the Terraform provisioning configuration** so the group can be re-created on demand without re-authoring the resource block

---

## Solution Architecture

The solution leverages Terraform's `terraform destroy -target` capability, which destroys a single named resource without affecting other resources tracked in the same state file. This approach provides:

* **Surgical precision**: Only the explicitly targeted resource is destroyed
* **State consistency**: Terraform state is updated atomically alongside the real infrastructure change
* **Configuration preservation**: The `.tf` file remains intact post-destroy, acting as a re-provisioning blueprint
* **Auditability**: The full plan/apply cycle produces terminal output that serves as a destruction audit trail

The working directory is `/home/bob/terraform` on the `iac-server` host.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Terraform | Initialized workspace with `.terraform/` directory present |
| AWS CLI | Configured and accessible for post-destroy verification |
| IAM permissions | Rights to delete IAM groups in the target account |
| Existing state | `aws_iam_group.this` tracked in `terraform.tfstate` |
| Provider configuration | `provider.tf` present and valid |

---

## Project Structure

```
/home/bob/terraform/
├── .terraform/                  # Provider plugins and module cache
├── .terraform.lock.hcl          # Provider version lock file
├── main.tf                      # IAM group resource definition (preserved post-destroy)
├── provider.tf                  # AWS provider configuration
├── terraform.tfstate            # Terraform state file (updated post-destroy)
├── terraform.tfstate.backup     # Automatic state backup created by Terraform
└── README.MD                    # Project readme
```

---

## Implementation Guide

### Step 1: Verify Working Directory and Project Files

Navigate into the Terraform working directory and inspect all present files to confirm the environment is correctly initialized before executing any destructive operations.

```bash
pwd
```

```
/home/bob/terraform
```

```bash
ls -la
```

```
total 40
drwxr-xr-x 1 bob bob 4096 May  3 04:51 .
drwxr-x--- 1 bob bob 4096 May  3 04:51 ..
drwxr-xr-x 3 bob bob 4096 May  3 04:51 .terraform
-rw-r--r-- 1 bob bob 1406 May  3 04:51 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob   60 May  3 04:51 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-rw-r-- 1 bob bob  747 May  3 04:51 terraform.tfstate
```

The `.terraform/` directory confirms the workspace is initialized. The `terraform.tfstate` file confirms an active state file is present.

*Screenshot: Terminal output showing `ls -la` in `/home/bob/terraform` with all listed files*

<img width="1149" height="526" alt="image" src="https://github.com/user-attachments/assets/f20d8dfd-3a9d-4cf4-8c08-25e5c1a2fbf9" />

---

### Step 2: Confirm the Target Resource in Configuration

Use `grep` to confirm the resource name `iamgroup_rose` is defined in `main.tf`, then display the full file contents to understand the complete resource block before proceeding.

```bash
grep -n "iamgroup_rose" main.tf
```

```
2:  name = "iamgroup_rose"
```

```bash
cat main.tf
```

```hcl
resource "aws_iam_group" "this" {
  name = "iamgroup_rose"
}
```

The resource address is `aws_iam_group.this`. This is the value that will be supplied to the `-target` flag.

*Screenshot: Terminal output showing `cat main.tf` with the IAM group resource block*

<img width="1132" height="649" alt="image" src="https://github.com/user-attachments/assets/2d4746dd-c9f1-4665-9623-538a2a0b259e" />

---

### Step 3: Inspect Current Terraform State

List all resources tracked in the current Terraform state file to confirm `aws_iam_group.this` is present and managed.

```bash
terraform state list
```

```
aws_iam_group.this
```

The resource exists in state and is eligible for a targeted destroy. An empty or missing state would require investigation before proceeding.

*Screenshot: Terminal output of `terraform state list` showing `aws_iam_group.this`*

<img width="1133" height="680" alt="image" src="https://github.com/user-attachments/assets/d1d439fe-86d4-4a24-8177-0929ab3c5bf5" />

---

### Step 4: Back Up the Configuration File

Create a copy of `main.tf` before the destroy operation as a safeguard, then use `diff` to confirm the backup is identical to the original.

```bash
cp main.tf main.tf.bak && diff main.tf main.tf.bak
```

The absence of any `diff` output confirms the files are byte-for-byte identical. This backup ensures the configuration is recoverable if the file is accidentally modified or deleted during the cleanup workflow.

*Screenshot: Terminal showing `cp` and `diff` commands with no diff output*

<img width="1134" height="694" alt="image" src="https://github.com/user-attachments/assets/88421922-4e7c-436b-9dd2-244b0b353624" />

---

### Step 5: Preview the Destroy Operation

Run a targeted destroy plan using the `-destroy` and `-target` flags combined to generate a preview of what Terraform will remove. No changes are applied at this stage.

```bash
terraform plan -destroy -target="aws_iam_group.this"
```

```
aws_iam_group.this: Refreshing state... [id=iamgroup_rose]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  - destroy

Terraform will perform the following actions:

  # aws_iam_group.this will be destroyed
  - resource "aws_iam_group" "this" {
      - arn       = "arn:aws:iam::000000000000:group/iamgroup_rose" -> null
      - id        = "iamgroup_rose" -> null
      - name      = "iamgroup_rose" -> null
      - path      = "/" -> null
      - unique_id = "cg6qdpt515j82n90n7dr" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

The plan output shows all five attributes of `aws_iam_group.this` being nullified. The `Plan: 0 to add, 0 to change, 1 to destroy` summary confirms exactly one resource will be removed and no unintended changes will occur.

Terraform also issues a `Warning: Resource targeting is in effect`, which is expected behavior when using `-target`. This warning does not indicate an error; it is an informational notice that the plan scope is limited to the targeted resource.

*Screenshot: Full output of `terraform plan -destroy -target="aws_iam_group.this"` showing the destruction plan and attribute nullification*

<img width="1147" height="675" alt="image" src="https://github.com/user-attachments/assets/821ba85a-22f8-46b1-9408-510ad2e30e51" />

---

### Step 6: Execute the Targeted Destroy

Apply the destroy operation, scoped to `aws_iam_group.this`. Terraform will re-evaluate the plan and prompt for explicit confirmation before removing the resource.

```bash
terraform destroy -target="aws_iam_group.this"
```

At the confirmation prompt, enter `yes` to authorize the destruction:

```
Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
```

```
aws_iam_group.this: Destroying... [id=iamgroup_rose]
aws_iam_group.this: Destruction complete after 0s

Destroy complete! Resources: 1 destroyed.
```

The `Destruction complete after 0s` output confirms the IAM group was removed from AWS and the Terraform state was updated synchronously.

*Screenshot: Full `terraform destroy` output including the confirmation prompt, `yes` response, and `Destroy complete! Resources: 1 destroyed.` summary*

<img width="1177" height="756" alt="image" src="https://github.com/user-attachments/assets/8f13c0d5-073c-4f64-926f-d25178934a82" />

---

### Step 7: Confirm State is Empty

Re-run `terraform state list` to verify that `aws_iam_group.this` has been removed from the state file and no other resources remain tracked.

```bash
terraform state list
```

The command returns no output, confirming the state file is now empty and Terraform no longer tracks any managed resources in this workspace.

*Screenshot: Terminal showing `terraform state list` with empty output after destroy*

<img width="1152" height="368" alt="image" src="https://github.com/user-attachments/assets/66dae38e-6882-48d3-a273-bb7fa4a1164b" />

---

### Step 8: Verify Deletion via AWS CLI

Use the AWS CLI to independently verify that the IAM group no longer exists in the AWS account, confirming the infrastructure change was applied successfully outside of Terraform's state layer.

```bash
aws iam get-group --group-name iamgroup_rose
```

```
An error occurred (NoSuchEntity) when calling the GetGroup operation: Group iamgroup_rose not found
```

The `NoSuchEntity` error from the IAM API is the expected response and confirms that `iamgroup_rose` has been fully deleted from AWS. This step validates that the Terraform state reflects actual infrastructure reality.

*Screenshot: AWS CLI output showing `NoSuchEntity` error confirming group deletion*

<img width="1152" height="512" alt="image" src="https://github.com/user-attachments/assets/5399000c-6cd5-4d68-8e2a-fea60a52b9f4" />

---

### Step 9: Validate Configuration Integrity Post-Destroy

Inspect `main.tf` to confirm the configuration file was not modified during the destroy operation, then run `terraform validate` to confirm the HCL is syntactically valid.

```bash
cat main.tf
```

```hcl
resource "aws_iam_group" "this" {
  name = "iamgroup_rose"
}
```

```bash
terraform validate
```

```
Success! The configuration is valid.
```

The configuration file is intact and unchanged. Terraform confirms the HCL is valid. The provisioning code is ready to be used for re-creating the IAM group at any future point.

*Screenshot: Terminal showing `cat main.tf` output and `terraform validate` returning `Success! The configuration is valid.`*

---

### Step 10: Run terraform plan to Confirm Re-Provisioning Readiness

Execute `terraform plan` without any destroy flags to confirm that the configuration is fully capable of re-creating the IAM group from scratch. This validates the preservation requirement.

```bash
terraform plan
```

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_group.this will be created
  + resource "aws_iam_group" "this" {
      + arn       = (known after apply)
      + id        = (known after apply)
      + name      = "iamgroup_rose"
      + path      = "/"
      + unique_id = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

The `Plan: 1 to add, 0 to change, 0 to destroy` output confirms that the configuration is coherent and fully re-deployable. Running `terraform apply` at any future point will recreate `iamgroup_rose` with identical configuration.

*Screenshot: Full `terraform plan` output showing `Plan: 1 to add, 0 to change, 0 to destroy` with the create action for `aws_iam_group.this`*

---

### Step 11: Clean Up the Backup File

Remove the temporary backup file created in Step 4, as the configuration file has been confirmed intact and the backup is no longer required.

```bash
rm main.tf.bak
```

*Screenshot: Terminal showing `rm main.tf.bak` command execution*

---

### Step 12: Final Directory Inspection

Run a final `ls -la` to confirm the directory state after the full operation. The backup file should be absent and a new `terraform.tfstate.backup` file should be present, created automatically by Terraform during the destroy operation.

```bash
ls -la
```

```
total 44
drwxr-xr-x 1 bob bob 4096 May  3 05:10 .
drwxr-x--- 1 bob bob 4096 May  3 04:51 ..
drwxr-xr-x 3 bob bob 4096 May  3 04:51 .terraform
-rw-r--r-- 1 bob bob 1406 May  3 04:51 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob   60 May  3 04:51 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-rw-r-- 1 bob bob  181 May  3 05:03 terraform.tfstate
-rw-r--r-- 1 bob bob  747 May  3 05:03 terraform.tfstate.backup
```

Key observations from the final state:

* `main.tf` is present and unchanged at 60 bytes, confirming configuration preservation
* `main.tf.bak` is absent, confirming successful cleanup
* `terraform.tfstate` is now 181 bytes (reduced from 747 bytes), reflecting an empty or near-empty state
* `terraform.tfstate.backup` is present at 747 bytes, which is Terraform's automatic backup of the pre-destroy state containing the last known attributes of `aws_iam_group.this`

*Screenshot: Terminal showing final `ls -la` output with `terraform.tfstate.backup` present and `main.tf.bak` absent*

---

## Best Practices

**Always preview before destroying**
Running `terraform plan -destroy -target=<resource>` before the actual destroy gives a full attribute-level view of what will be removed. This eliminates ambiguity and reduces the risk of accidental data loss.

**Use `-target` for surgical operations only**
The `-target` flag is intentionally not designed for routine workflow. It is the correct tool for one-off deprovisioning, error recovery, or migration cleanup. Using it in a standard apply cycle bypasses dependency resolution and can leave state in a partially applied condition.

**Back up the configuration before destructive operations**
Creating a `cp main.tf main.tf.bak` backup before any destroy and verifying with `diff` adds a low-cost safety net. Remove the backup only after confirming the configuration file is intact.

**Validate state before and after every destroy**
Running `terraform state list` before and after the destroy confirms the operation had the exact intended scope. An unexpected entry in the post-destroy state list signals that something outside the target was affected.

**Cross-verify with the AWS CLI**
Terraform state is authoritative, but verifying resource deletion via `aws iam get-group` (or equivalent service APIs) provides independent confirmation that the infrastructure change was applied to the real environment, not just updated in state.

**Preserve configuration for re-provisioning**
When a resource is being removed temporarily rather than permanently, retaining the `.tf` file with the original resource block allows the team to re-create the infrastructure with a single `terraform apply` without re-authoring. Running `terraform plan` after the destroy validates this re-provisioning capability.

**Acknowledge and understand the targeting warning**
Terraform issues a `Warning: Resource targeting is in effect` during every `-target` operation. This is not an error. Understanding that the warning exists to signal limited plan scope prevents confusion during team handoff or incident review.

**Review `terraform.tfstate.backup` retention policy**
Terraform automatically creates `terraform.tfstate.backup` during state-modifying operations. This file contains the last known good state before the operation. In production workflows, this file should be retained in a version-controlled or remote backend (S3, Terraform Cloud) rather than on local disk.

---

## Errors and Resolutions

### Error: `NoSuchEntity` from AWS IAM API (Expected)

**Command:**
```bash
aws iam get-group --group-name iamgroup_rose
```

**Output:**
```
An error occurred (NoSuchEntity) when calling the GetGroup operation: Group iamgroup_rose not found
```

**Root Cause:**
This is not an operational error. The `NoSuchEntity` response from the AWS IAM API is the expected and correct output after a successful group deletion. The AWS CLI's `get-group` operation returns this error code when the specified group does not exist in the account.

**Resolution:**
No action required. This output is the verification signal that confirms `iamgroup_rose` was successfully removed from AWS and the destroy operation completed as intended.

---

### Warning: Resource Targeting is in Effect

**Context:**
This warning appears during both `terraform plan -destroy -target` and `terraform destroy -target` operations:

```
Warning: Resource targeting is in effect

You are creating a plan with the -target option, which means that the result of this plan may not represent all of the changes
requested by the current configuration.
```

**Root Cause:**
Terraform issues this warning whenever the `-target` flag is used to signal that the plan or apply is scoped to a subset of the configuration. Resources with dependencies on the targeted resource may not be evaluated, and output values may not reflect the full configuration state.

**Resolution:**
In this implementation, the workspace contains a single resource (`aws_iam_group.this`) with no dependent resources or outputs. The warning is expected, accurate, and does not represent a problem. After the targeted destroy, running `terraform plan` without any target confirmed no other resources are pending.

---

## Lessons Learned

**Targeted destroy is not `terraform destroy` on a single resource by default**
A common assumption is that `terraform destroy` without flags will prompt for confirmation per resource. In practice, it destroys everything in state. The `-target` flag is the mechanism for scoping destruction to a single resource address. This distinction is critical in multi-resource workspaces where running an untargeted destroy would remove unrelated infrastructure.

**The plan before destroy is not optional in production**
Running `terraform plan -destroy -target` before the actual `terraform destroy -target` may seem redundant since Terraform re-plans during apply anyway. In practice, reviewing the plan output in isolation allows engineers to inspect attribute values (ARN, unique ID, path) of the resource being removed and catch mismatches before they result in real infrastructure changes.

**Configuration preservation is a first-class concern**
Infrastructure code and infrastructure state are separate entities. Destroying a resource removes it from state and from AWS, but it does not and should not remove the configuration that defined it. Explicitly validating that `main.tf` is intact after the destroy, and confirming re-provisioning readiness with `terraform plan`, transforms a cleanup task into a reversible operation.

**`terraform.tfstate.backup` is an automatic safety net, not a substitute for remote state**
Terraform automatically writes the pre-operation state to `terraform.tfstate.backup` during any state-modifying run. While useful for local recovery, this file is stored on the same machine as the primary state and is overwritten on every subsequent operation. In production environments, remote backends (S3 with versioning, Terraform Cloud) provide durable, versioned state history that survives accidental local file loss.

**Verifying with the AWS CLI closes the feedback loop**
Terraform's state confirms what Terraform believes to be true about the infrastructure. The AWS CLI confirms what is actually true in the AWS control plane. Running `aws iam get-group` after the destroy validates that Terraform's state reflects reality and that no API-level error silently prevented deletion while state was still updated.

**Account ID `000000000000` identifies a LocalStack environment**
The ARN `arn:aws:iam::000000000000:group/iamgroup_rose` in the plan output uses account ID `000000000000`, which is the default placeholder used by LocalStack for simulated AWS environments. All commands and behaviors documented here are directly transferable to real AWS accounts where the actual 12-digit account ID would appear instead.










<img width="1150" height="479" alt="image" src="https://github.com/user-attachments/assets/077a334a-cd33-4801-9d16-119374539f8c" />
<img width="1148" height="554" alt="image" src="https://github.com/user-attachments/assets/1a21b618-e6e6-4b32-8047-2bc92ec47017" />
<img width="1151" height="584" alt="image" src="https://github.com/user-attachments/assets/24a4f6fe-d6f2-41e8-981c-1fa4de7fa46c" />
<img width="1149" height="480" alt="image" src="https://github.com/user-attachments/assets/19ee31c8-7487-4f6e-9a0e-7ae897ab934b" />
<img width="1148" height="699" alt="image" src="https://github.com/user-attachments/assets/3148210b-e19b-444d-849a-a2b2d2cd512e" />





