# Terraform IAM Policy Attachment to IAM User on AWS

Attaching a managed IAM policy to an existing IAM user using Terraform infrastructure-as-code, executed against an AWS environment via LocalStack-emulated endpoints. This implementation extends an existing Terraform configuration without introducing new resource files, preserving project structure discipline and configuration cohesion.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Problem Statement](#problem-statement)
* [Solution Summary](#solution-summary)
* [Architecture and Design Intent](#architecture-and-design-intent)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify the Working Directory and Existing Configuration](#step-1-verify-the-working-directory-and-existing-configuration)
  * [Step 2: Review the Existing main.tf Configuration](#step-2-review-the-existing-maintf-configuration)
  * [Step 3: Append the Policy Attachment Resource to main.tf](#step-3-append-the-policy-attachment-resource-to-maintf)
  * [Step 4: Verify the Updated main.tf Configuration](#step-4-verify-the-updated-maintf-configuration)
  * [Step 5: Validate the Terraform Configuration](#step-5-validate-the-terraform-configuration)
  * [Step 6: Generate and Review the Execution Plan](#step-6-generate-and-review-the-execution-plan)
  * [Step 7: Apply the Configuration](#step-7-apply-the-configuration)
  * [Step 8: Confirm Policy Attachment via AWS CLI](#step-8-confirm-policy-attachment-via-aws-cli)
* [Validation and Output](#validation-and-output)
* [Key Engineering Decisions](#key-engineering-decisions)
* [Best Practices Applied](#best-practices-applied)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)
* [References](#references)

---

## Project Overview

| Attribute | Detail |
|---|---|
| **Cloud Provider** | AWS (LocalStack-emulated) |
| **IaC Tool** | Terraform |
| **Resource Type** | `aws_iam_user_policy_attachment` |
| **IAM User** | `iamuser_kirsty` |
| **IAM Policy** | `iampolicy_kirsty` |
| **Working Directory** | `/home/bob/terraform` |
| **Modified File** | `main.tf` (in-place append, no new file created) |

---

## Problem Statement

The Nautilus DevOps team is executing a phased AWS cloud migration for multiple services. To maintain fine-grained access control throughout the migration, each IAM identity must be granted only the permissions required for its workload scope.

An IAM user named `iamuser_kirsty` and a customer-managed IAM policy named `iampolicy_kirsty` had been provisioned in a prior Terraform apply cycle. However, the policy had not been attached to the user, leaving the user without any effective permissions. The requirement was to attach `iampolicy_kirsty` to `iamuser_kirsty` using Terraform, within the existing `main.tf` file, without creating additional `.tf` files.

---

## Solution Summary

An `aws_iam_user_policy_attachment` resource block was appended to the existing `main.tf` using a heredoc append operation. Terraform's `validate`, `plan`, and `apply` sequence was then used to attach the policy to the user. Final verification was performed using the AWS CLI to confirm the attachment at the IAM service level.

---

## Architecture and Design Intent

```
iamuser_kirsty (aws_iam_user)
        |
        | aws_iam_user_policy_attachment
        |
iampolicy_kirsty (aws_iam_policy)
  - Effect: Allow
  - Action: ec2:Read*
  - Resource: *
```

The policy grants read-only EC2 access to the user. Using `aws_iam_user_policy_attachment` (as opposed to inline policies) follows AWS IAM best practices by keeping policy definitions decoupled from user resources, enabling policy reuse and centralized access auditing.

---

## Prerequisites

* Terraform initialized in `/home/bob/terraform` (`.terraform/` directory and `.terraform.lock.hcl` present)
* AWS provider configured in `provider.tf`
* `iamuser_kirsty` and `iampolicy_kirsty` already provisioned and tracked in `terraform.tfstate`
* AWS CLI configured and accessible
* Sufficient IAM permissions to attach policies to users

---

## Repository Structure

```
/home/bob/terraform/
├── .terraform/                  # Provider plugins (initialized)
├── .terraform.lock.hcl          # Provider dependency lock file
├── main.tf                      # Primary resource definitions (modified in this implementation)
├── provider.tf                  # AWS provider configuration
├── terraform.tfstate            # Current state file
└── README.MD                    # Project readme
```

---

## Implementation Guide

### Step 1: Verify the Working Directory and Existing Configuration

Confirm the Terraform working directory is `/home/bob/terraform` and that all expected files are present before making any changes.

```bash
pwd
```

```bash
ls -la
```

**Expected output:** The working directory contains `main.tf`, `provider.tf`, `.terraform/`, `.terraform.lock.hcl`, and `terraform.tfstate`. The presence of `.terraform/` confirms the workspace is already initialized.

*Screenshot: Directory listing showing all Terraform project files in /home/bob/terraform*

<img width="1083" height="544" alt="image" src="https://github.com/user-attachments/assets/28b3f934-28fb-4c00-9129-0074e5c5ff15" />

---

### Step 2: Review the Existing main.tf Configuration

Inspect the current contents of `main.tf` to understand the two resources already defined before appending the attachment resource.

```bash
cat main.tf
```

**Existing configuration:**

```hcl
# Create IAM user
resource "aws_iam_user" "user" {
  name = "iamuser_kirsty"

  tags = {
    Name = "iamuser_kirsty"
  }
}

# Create IAM Policy
resource "aws_iam_policy" "policy" {
  name        = "iampolicy_kirsty"
  description = "IAM policy allowing EC2 read actions for kirsty"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ec2:Read*"]
        Resource = "*"
      }
    ]
  })
}
```

Both `aws_iam_user.user` and `aws_iam_policy.policy` are present and tracked in state. The missing piece is the attachment resource binding them together.

*Screenshot: Terminal output of cat main.tf showing the existing IAM user and policy resource blocks*

<img width="1079" height="748" alt="image" src="https://github.com/user-attachments/assets/081a17b3-30af-4bde-a18c-d06c89206eba" />

---

### Step 3: Append the Policy Attachment Resource to main.tf

Append the `aws_iam_user_policy_attachment` resource block directly to `main.tf` using a heredoc append. This approach modifies the file in-place without creating a separate `.tf` file, satisfying the project's structural constraint.

```bash
cat >> main.tf << 'EOF'

# Attach IAM policy to IAM user
resource "aws_iam_user_policy_attachment" "kirsty_policy_attachment" {
  user       = aws_iam_user.user.name
  policy_arn = aws_iam_policy.policy.arn
}
EOF
```

The resource references `aws_iam_user.user.name` and `aws_iam_policy.policy.arn` using Terraform's implicit dependency resolution. This ensures Terraform understands the relationship between these resources and applies them in the correct order.

*Screenshot: Terminal showing the heredoc append command being executed against main.tf*

---

### Step 4: Verify the Updated main.tf Configuration

Confirm the append was successful and that the full file contents are syntactically correct before running any Terraform commands.

```bash
cat main.tf
```

**Expected full content after append:**

```hcl
# Create IAM user
resource "aws_iam_user" "user" {
  name = "iamuser_kirsty"

  tags = {
    Name = "iamuser_kirsty"
  }
}

# Create IAM Policy
resource "aws_iam_policy" "policy" {
  name        = "iampolicy_kirsty"
  description = "IAM policy allowing EC2 read actions for kirsty"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["ec2:Read*"]
        Resource = "*"
      }
    ]
  })
}
# Attach IAM policy to IAM user
resource "aws_iam_user_policy_attachment" "kirsty_policy_attachment" {
  user       = aws_iam_user.user.name
  policy_arn = aws_iam_policy.policy.arn
}
```

*Screenshot: Terminal output of cat main.tf showing the complete configuration including the newly appended attachment block*

---

### Step 5: Validate the Terraform Configuration

Run `terraform validate` to confirm the HCL syntax and resource schema are valid before generating a plan.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

*Screenshot: Terminal output showing terraform validate returning Success! The configuration is valid.*

---

### Step 6: Generate and Review the Execution Plan

Run `terraform plan` to preview the changes Terraform will make. This step allows review of the planned action before any infrastructure is modified.

```bash
terraform plan
```

**Expected plan output:**

```
aws_iam_policy.policy: Refreshing state... [id=arn:aws:iam::000000000000:policy/iampolicy_kirsty]
aws_iam_user.user: Refreshing state... [id=iamuser_kirsty]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_user_policy_attachment.kirsty_policy_attachment will be created
  + resource "aws_iam_user_policy_attachment" "kirsty_policy_attachment" {
      + id         = (known after apply)
      + policy_arn = "arn:aws:iam::000000000000:policy/iampolicy_kirsty"
      + user       = "iamuser_kirsty"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Terraform correctly identifies the two pre-existing resources from state and determines that only the attachment resource requires creation. No destructive changes are planned.

*Screenshot: Terminal output of terraform plan showing the single resource to be created with policy ARN and user name resolved*

---

### Step 7: Apply the Configuration

Apply the configuration using the `-auto-approve` flag to bypass the interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Expected apply output:**

```
aws_iam_user.user: Refreshing state... [id=iamuser_kirsty]
aws_iam_policy.policy: Refreshing state... [id=arn:aws:iam::000000000000:policy/iampolicy_kirsty]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  + create

Terraform will perform the following actions:

  # aws_iam_user_policy_attachment.kirsty_policy_attachment will be created
  + resource "aws_iam_user_policy_attachment" "kirsty_policy_attachment" {
      + id         = (known after apply)
      + policy_arn = "arn:aws:iam::000000000000:policy/iampolicy_kirsty"
      + user       = "iamuser_kirsty"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_iam_user_policy_attachment.kirsty_policy_attachment: Creating...
aws_iam_user_policy_attachment.kirsty_policy_attachment: Creation complete after 0s [id=iamuser_kirsty-20260429014339913900000001]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The attachment resource was created successfully with a system-assigned ID of `iamuser_kirsty-20260429014339913900000001`.

*Screenshot: Terminal output of terraform apply -auto-approve showing Apply complete! Resources: 1 added, 0 changed, 0 destroyed.*

---

### Step 8: Confirm Policy Attachment via AWS CLI

Use the AWS CLI to independently verify that `iampolicy_kirsty` is now listed as an attached policy on `iamuser_kirsty`. This step validates the attachment at the IAM service level, outside of Terraform state.

```bash
aws iam list-attached-user-policies --user-name iamuser_kirsty
```

**Expected output:**

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_kirsty",
            "PolicyArn": "arn:aws:iam::000000000000:policy/iampolicy_kirsty"
        }
    ]
}
```

The policy is confirmed attached to the user at the IAM level. The `PolicyArn` matches the ARN resolved during the Terraform apply.

*Screenshot: Terminal output of aws iam list-attached-user-policies showing iampolicy_kirsty attached to iamuser_kirsty*

---

## Validation and Output

| Validation Check | Method | Result |
|---|---|---|
| HCL syntax and schema validity | `terraform validate` | Passed |
| Planned resource delta | `terraform plan` | 1 resource to add, 0 to change, 0 to destroy |
| Apply execution | `terraform apply -auto-approve` | 1 resource added successfully |
| IAM attachment confirmed | `aws iam list-attached-user-policies` | `iampolicy_kirsty` attached to `iamuser_kirsty` |

---

## Key Engineering Decisions

* **In-place file append over new resource file:** The task constraint required modifying `main.tf` directly rather than creating a separate `.tf` file. A heredoc append (`cat >> main.tf << 'EOF'`) was used, which is idempotent in intent and avoids editor dependency, making it suitable for terminal-only environments.

* **Implicit resource references over hardcoded values:** The attachment block uses `aws_iam_user.user.name` and `aws_iam_policy.policy.arn` rather than literal strings. This establishes Terraform's implicit dependency graph, ensuring the user and policy are resolved before the attachment is created.

* **Customer-managed policy over inline policy:** Using `aws_iam_user_policy_attachment` with a separately defined `aws_iam_policy` resource follows AWS best practices. Managed policies can be versioned, reused across multiple identities, and audited independently.

* **Plan before apply discipline:** Even with `-auto-approve` used in the final apply, `terraform plan` was executed first to review the planned diff and confirm no unintended changes would occur to the existing user or policy resources.

* **CLI verification as a secondary source of truth:** Verification was performed using `aws iam list-attached-user-policies` rather than relying solely on Terraform state. This confirms the attachment exists in the actual AWS (LocalStack) service layer, not just in `.tfstate`.

---

## Best Practices Applied

* **Least privilege IAM design:** The policy grants only `ec2:Read*` actions, limiting the user to read-only EC2 access. No write, create, delete, or administrative EC2 permissions are included.

* **Validate before plan before apply:** The full `validate > plan > apply` sequence was followed, catching any configuration issues early without incurring API calls or infrastructure changes.

* **Separation of identity and permissions:** The IAM user, IAM policy, and their attachment are defined as three distinct resource blocks. This separation makes individual resources independently manageable and auditable.

* **Descriptive resource naming:** Resource logical names (`kirsty_policy_attachment`) and physical names (`iampolicy_kirsty`, `iamuser_kirsty`) are consistent and reflect the identity they serve, reducing ambiguity in state files and plan output.

* **State-aware refresh:** Terraform's pre-plan state refresh confirmed that both `aws_iam_user.user` and `aws_iam_policy.policy` were already in state and did not require recreation, protecting existing resources from unintended replacement.

* **Independent CLI verification:** Post-apply verification using the AWS CLI ensures the infrastructure behaves as expected at the service level, independent of Terraform's internal state representation.

---

## Errors and Resolutions

No errors were encountered during this implementation. The configuration validated cleanly, the plan produced the expected single-resource diff, and the apply completed successfully in under one second.

The following failure modes were anticipated and avoided through the approach taken:

| Anticipated Risk | Mitigation Applied |
|---|---|
| Duplicate resource error if a new `.tf` file was created alongside `main.tf` with overlapping resource names | Appended directly to `main.tf` as required, no additional file created |
| Hardcoded ARN drift if the policy ARN changed | Used `aws_iam_policy.policy.arn` reference instead of a literal ARN string |
| Unintended replacement of existing resources | Pre-apply plan review confirmed only 1 addition with 0 changes and 0 destructions |

---

## Lessons Learned

* **Heredoc append is a reliable terminal-native file modification technique.** When working in environments without a GUI editor, `cat >> file << 'EOF' ... EOF` is a clean, readable method for appending structured content. Single-quoting `'EOF'` prevents shell variable expansion inside the block, which is critical when the content contains Terraform interpolation syntax.

* **Terraform's implicit dependency resolution removes the need for explicit `depends_on` in straightforward attachment patterns.** Because the attachment resource references the user and policy by their Terraform resource attributes rather than hardcoded values, Terraform automatically constructs the correct apply order without additional configuration.

* **Verifying attachments at the IAM service level, not just in Terraform state, is a production habit worth building.** Terraform state can in rare scenarios diverge from actual infrastructure. Confirming the attachment with `aws iam list-attached-user-policies` validates the end state independently and is the kind of verification that belongs in any runbook or deployment checklist.

* **Customer-managed policies should always be preferred over inline policies in team environments.** Inline policies are user-scoped and cannot be reused or independently version-controlled. Managed policies can be attached to multiple users, roles, or groups and are visible in IAM policy listing, making access auditing significantly easier at scale.

* **Keeping all resources in a single `main.tf` for small configurations is a valid structural choice, but as configuration grows, decomposing into `iam.tf`, `networking.tf`, and similar domain-specific files improves readability and team collaboration.**

---

## References

* [Terraform AWS Provider: aws_iam_user_policy_attachment](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user_policy_attachment)
* [Terraform AWS Provider: aws_iam_user](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_user)
* [Terraform AWS Provider: aws_iam_policy](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/iam_policy)
* [AWS CLI Reference: list-attached-user-policies](https://docs.aws.amazon.com/cli/latest/reference/iam/list-attached-user-policies.html)
* [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
* [Terraform Command: validate](https://developer.hashicorp.com/terraform/cli/commands/validate)
* [Terraform Command: plan](https://developer.hashicorp.com/terraform/cli/commands/plan)
* [Terraform Command: apply](https://developer.hashicorp.com/terraform/cli/commands/apply)





<img width="1079" height="557" alt="image" src="https://github.com/user-attachments/assets/bc9a55a1-d24c-480b-821b-6f593ad19dac" />
<img width="1088" height="606" alt="image" src="https://github.com/user-attachments/assets/f68a967a-2289-44f5-93da-ae15b0e8523f" />
<img width="1090" height="681" alt="image" src="https://github.com/user-attachments/assets/21578620-2f86-4e5c-a3ac-9a1838ceb98e" />
<img width="1092" height="532" alt="image" src="https://github.com/user-attachments/assets/7a343367-2df4-45b4-b80f-92ceca1df81c" />
<img width="1083" height="566" alt="image" src="https://github.com/user-attachments/assets/b2a0470f-b6ea-4cb9-9e07-03419b935b3b" />
<img width="1092" height="627" alt="image" src="https://github.com/user-attachments/assets/d8c91340-5923-437f-a1c6-a5e610479779" />


