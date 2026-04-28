# Terraform: AWS Elastic IP Association to EC2 Instance

Attach a pre-provisioned Elastic IP to an existing EC2 instance in AWS using Terraform infrastructure-as-code by extending an existing `main.tf` configuration with an `aws_eip_association` resource block, then validating the binding through the AWS CLI.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Environment Details](#environment-details)
* [Implementation](#implementation)
  * [Step 1: Verify Existing Infrastructure State](#step-1-verify-existing-infrastructure-state)
  * [Step 2: Confirm EIP Is Unassociated](#step-2-confirm-eip-is-unassociated)
  * [Step 3: Review Existing Terraform Configuration](#step-3-review-existing-terraform-configuration)
  * [Step 4: Append the EIP Association Resource Block](#step-4-append-the-eip-association-resource-block)
  * [Step 5: Verify Updated Configuration](#step-5-verify-updated-configuration)
  * [Step 6: Format and Validate](#step-6-format-and-validate)
  * [Step 7: Plan the Change](#step-7-plan-the-change)
  * [Step 8: Apply the Configuration](#step-8-apply-the-configuration)
  * [Step 9: Confirm Association via AWS CLI](#step-9-confirm-association-via-aws-cli)
  * [Step 10: Verify Terraform State](#step-10-verify-terraform-state)
* [Key Decisions](#key-decisions)
* [Errors and Resolutions](#errors-and-resolutions)
* [Lessons Learned](#lessons-learned)
* [Reference](#reference)

---

## Project Overview

**Context:** The Nautilus DevOps team is executing a staged AWS cloud migration, decomposing the full migration into discrete infrastructure tasks to reduce risk and improve traceability. Each task targets a single infrastructure concern, enabling independent validation before proceeding.

**Problem Statement:** A pre-provisioned EC2 instance (`devops-ec2`) and a pre-provisioned Elastic IP (`devops-ec2-eip`) exist in `us-east-1` but are not yet associated. The EIP must be bound to the instance exclusively through Terraform, with the binding defined in the existing `main.tf` file. No separate `.tf` files are permitted.

**Solution:** Extend the existing `main.tf` with an `aws_eip_association` resource that references both the EC2 instance and EIP using Terraform resource attribute references, then apply the change and validate the association via AWS CLI.

---

## Architecture

```
us-east-1
+----------------------------------------------------------+
|  VPC                                                     |
|  +----------------------------------------------------+  |
|  |  Subnet: subnet-09fd8336f33a07d83                  |  |
|  |                                                    |  |
|  |  +----------------------------------------------+ |  |
|  |  |  EC2 Instance: devops-ec2                    | |  |
|  |  |  ID: i-904804de1ba0e9698                     | |  |
|  |  |  AMI: ami-0c101f26f147fa7fd                  | |  |
|  |  |  Type: t2.micro                              | |  |
|  |  |  SG: sg-f6c610043a019b043                    | |  |
|  |  |                                              | |  |
|  |  |  Elastic IP Association                      | |  |
|  |  |  AssocID: eipassoc-ba991b707fd6e24b6         | |  |
|  |  |                                              | |  |
|  |  |  EIP: devops-ec2-eip                         | |  |
|  |  |  AllocID: eipalloc-f97a52f19055b1a35         | |  |
|  |  |  Public IP: 127.188.198.172                  | |  |
|  |  +----------------------------------------------+ |  |
|  +----------------------------------------------------+  |
+----------------------------------------------------------+

Terraform State (terraform.tfstate)
  aws_instance.ec2
  aws_eip.ec2_eip
  aws_eip_association.devops_eip_assoc   <-- added this project
```

---

## Prerequisites

* AWS CLI configured with credentials and access to `us-east-1`
* Terraform v1.11.0 or later installed
* Working directory initialized (`terraform init` already executed, `.terraform/providers` present)
* Pre-existing resources in AWS state:
  * EC2 instance tagged `Name=devops-ec2`
  * Elastic IP tagged `Name=devops-ec2-eip`
* Write access to `/home/bob/terraform/main.tf`

---

## Environment Details

| Parameter | Value |
|---|---|
| Terraform Version | v1.11.0 |
| AWS Provider Version | hashicorp/aws v5.91.0 |
| Region | us-east-1 |
| Working Directory | `/home/bob/terraform` |
| EC2 Instance ID | `i-904804de1ba0e9698` |
| EC2 Instance Name Tag | `devops-ec2` |
| EIP Allocation ID | `eipalloc-f97a52f19055b1a35` |
| EIP Name Tag | `devops-ec2-eip` |
| AMI | `ami-0c101f26f147fa7fd` |
| Instance Type | `t2.micro` |
| Subnet | `subnet-09fd8336f33a07d83` |
| Security Group | `sg-f6c610043a019b043` |

---

## Implementation

### Step 1: Verify Existing Infrastructure State

Before modifying any Terraform configuration, confirm the real IDs of the pre-provisioned resources using the AWS CLI. This prevents configuration drift and ensures resource references are valid.

Retrieve the EC2 instance ID by filtering on the `Name` tag:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=devops-ec2" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text
```

**Output:**

```
i-904804de1ba0e9698
```

Retrieve the EIP allocation ID by filtering on the `Name` tag:

```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=devops-ec2-eip" \
  --query "Addresses[0].AllocationId" \
  --output text
```

**Output:**

```
eipalloc-f97a52f19055b1a35
```

> Screenshot: AWS CLI output confirming EC2 instance ID `i-904804de1ba0e9698` and EIP allocation ID `eipalloc-f97a52f19055b1a35`

<img width="1048" height="532" alt="image" src="https://github.com/user-attachments/assets/c1a4b394-06b3-4b1b-9fef-602da95025b7" />

---

### Step 2: Confirm EIP Is Unassociated

Verify that the EIP has no existing association before proceeding. This guards against duplicate association errors during `terraform apply`.

```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=devops-ec2-eip" \
  --query "Addresses[0].AssociationId" \
  --output text
```

**Output:**

```
None
```

The `None` response confirms the EIP is allocated but not yet associated with any instance.

> Screenshot: AWS CLI output showing `AssociationId` as `None`, confirming the EIP is free to associate

<img width="1044" height="592" alt="image" src="https://github.com/user-attachments/assets/49e7a5cd-5df7-465b-a50d-b408eeb35cba" />

---

### Step 3: Review Existing Terraform Configuration

Inspect the current `main.tf` to understand the existing resource blocks before appending a new one.

```bash
cat main.tf
```

**Output:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  subnet_id     = "subnet-09fd8336f33a07d83"
  vpc_security_group_ids = [
    "sg-f6c610043a019b043"
  ]

  tags = {
    Name = "devops-ec2"
  }
}

# Provision Elastic IP
resource "aws_eip" "ec2_eip" {
  tags = {
    Name = "devops-ec2-eip"
  }
}
```

Two resources are already defined: `aws_instance.ec2` and `aws_eip.ec2_eip`. Both are present in Terraform state, confirmed by the `Refreshing state...` messages produced during later plan and apply operations.

> Screenshot: Terminal output of `cat main.tf` showing the two existing resource blocks

---

### Step 4: Append the EIP Association Resource Block

Append the `aws_eip_association` resource to `main.tf` using a heredoc to avoid manual editor errors and ensure exact formatting. The resource uses Terraform attribute references (`aws_instance.ec2.id` and `aws_eip.ec2_eip.id`) rather than hardcoded IDs, preserving configuration portability and dependency graph correctness.

```bash
cat >> main.tf << 'EOF'

# Associate Elastic IP with EC2 instance
resource "aws_eip_association" "devops_eip_assoc" {
  instance_id   = aws_instance.ec2.id
  allocation_id = aws_eip.ec2_eip.id
}
EOF
```

No output on success. The heredoc appends the block directly to the end of the existing file.

> Screenshot: Terminal showing the `cat >>` heredoc command executed without error

---

### Step 5: Verify Updated Configuration

Confirm the full `main.tf` content after appending to validate the file is structurally complete and no content was lost or duplicated.

```bash
cat main.tf
```

**Output:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  subnet_id     = "subnet-09fd8336f33a07d83"
  vpc_security_group_ids = [
    "sg-f6c610043a019b043"
  ]

  tags = {
    Name = "devops-ec2"
  }
}

# Provision Elastic IP
resource "aws_eip" "ec2_eip" {
  tags = {
    Name = "devops-ec2-eip"
  }
}

# Associate Elastic IP with EC2 instance
resource "aws_eip_association" "devops_eip_assoc" {
  instance_id   = aws_instance.ec2.id
  allocation_id = aws_eip.ec2_eip.id
}
```

All three resource blocks are present and correctly structured.

> Screenshot: Full `cat main.tf` output showing all three resource blocks including the newly appended `aws_eip_association`

---

### Step 6: Format and Validate

Run `terraform fmt` to enforce canonical HCL formatting. This resolves any indentation inconsistencies introduced by the heredoc append and ensures the file conforms to Terraform style standards.

```bash
terraform fmt
```

**Output:**

```
provider.tf
```

Terraform reports `provider.tf` as reformatted. The `main.tf` file required no formatting changes, confirming the heredoc produced properly aligned HCL.

Validate the configuration for syntax and provider schema compliance:

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

> Screenshot: Terminal output of `terraform fmt` and `terraform validate` confirming valid configuration

---

### Step 7: Plan the Change

Generate an execution plan to preview the exact actions Terraform will take. This confirms Terraform correctly reads both existing resources from state and identifies only the association as a new resource to create.

```bash
terraform plan
```

**Output:**

```
aws_eip.ec2_eip: Refreshing state... [id=eipalloc-f97a52f19055b1a35]
aws_instance.ec2: Refreshing state... [id=i-904804de1ba0e9698]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_eip_association.devops_eip_assoc will be created
  + resource "aws_eip_association" "devops_eip_assoc" {
      + allocation_id        = "eipalloc-f97a52f19055b1a35"
      + id                   = (known after apply)
      + instance_id          = "i-904804de1ba0e9698"
      + network_interface_id = (known after apply)
      + private_ip_address   = (known after apply)
      + public_ip            = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

The plan shows:

* `aws_eip.ec2_eip` and `aws_instance.ec2` are refreshed from state, no changes detected
* Only `aws_eip_association.devops_eip_assoc` is scheduled for creation
* Zero destructive actions

> Screenshot: `terraform plan` output showing `Plan: 1 to add, 0 to change, 0 to destroy`

---

### Step 8: Apply the Configuration

Execute the plan and confirm the apply prompt with `yes` to create the EIP association.

```bash
terraform apply
```

At the confirmation prompt:

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes
```

**Output:**

```
aws_eip_association.devops_eip_assoc: Creating...
aws_eip_association.devops_eip_assoc: Creation complete after 0s [id=eipassoc-ba991b707fd6e24b6]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

The association was created in under one second. The association ID `eipassoc-ba991b707fd6e24b6` is now tracked in Terraform state.

> Screenshot: `terraform apply` completion output showing `Apply complete! Resources: 1 added, 0 changed, 0 destroyed` and association ID `eipassoc-ba991b707fd6e24b6`

---

### Step 9: Confirm Association via AWS CLI

Query the EIP record directly from AWS to independently verify the association outside of Terraform state. This cross-validates the infrastructure change against the actual AWS control plane.

```bash
aws ec2 describe-addresses \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=devops-ec2-eip" \
  --query "Addresses[0].{PublicIP:PublicIp,InstanceId:InstanceId,AssociationId:AssociationId}" \
  --output table
```

**Output:**

```
--------------------------------------------------------------------------
|                            DescribeAddresses                           |
+----------------------------+-----------------------+-------------------+
|        AssociationId       |      InstanceId       |     PublicIP      |
+----------------------------+-----------------------+-------------------+
|  eipassoc-ba991b707fd6e24b6|  i-904804de1ba0e9698  |  127.188.198.172  |
+----------------------------+-----------------------+-------------------+
```

All three fields are populated:

| Field | Value |
|---|---|
| AssociationId | `eipassoc-ba991b707fd6e24b6` |
| InstanceId | `i-904804de1ba0e9698` |
| PublicIP | `127.188.198.172` |

The EIP is now bound to the correct instance.

> Screenshot: AWS CLI table output confirming EIP association with `AssociationId`, `InstanceId`, and `PublicIP` all populated

---

### Step 10: Verify Terraform State

Confirm that all three resources are tracked in Terraform state after the apply.

```bash
terraform state list
```

**Output:**

```
aws_eip.ec2_eip
aws_eip_association.devops_eip_assoc
aws_instance.ec2
```

All three managed resources are present in state. Terraform will manage their full lifecycle going forward, including detection of any out-of-band changes.

> Screenshot: `terraform state list` output showing all three resources tracked

---

## Key Decisions

**Terraform attribute references over hardcoded IDs**

The `aws_eip_association` resource uses `aws_instance.ec2.id` and `aws_eip.ec2_eip.id` instead of the literal IDs discovered via AWS CLI. This creates an implicit dependency graph so Terraform correctly sequences creation order and keeps the configuration portable across environments where IDs differ.

**Single `main.tf` file constraint respected**

The task required all changes to be made within `main.tf`. No new `.tf` files were created. This is a common real-world constraint in environments where Terraform configurations are maintained by policy as single-file modules for simplicity and auditability.

**Heredoc append over in-editor modification**

Using `cat >> main.tf << 'EOF' ... EOF` appended the new block atomically from the terminal without requiring a text editor session, reducing the risk of accidental file corruption or whitespace errors.

**Pre-apply CLI verification**

Querying the EIP association ID and instance ID via AWS CLI before modifying Terraform configuration confirmed the expected pre-state and served as a baseline for post-apply validation.

**Post-apply dual validation**

Both `aws ec2 describe-addresses` and `terraform state list` were used after apply. CLI validation confirms the real AWS resource state; state list validation confirms Terraform's internal model is consistent with that state.

---

## Errors and Resolutions

No errors were encountered during this implementation. The execution proceeded cleanly through all stages: resource ID discovery, configuration extension, format, validate, plan, and apply.

**Potential failure mode documented for reference:**

| Scenario | Symptom | Root Cause | Resolution |
|---|---|---|---|
| EIP already associated | `terraform apply` fails with `InvalidIPAddress.InUse` | The EIP was manually associated in the console before Terraform apply | Disassociate the EIP via AWS CLI or console, then re-run `terraform apply` |
| Resource not in Terraform state | `terraform plan` shows `aws_instance.ec2` and `aws_eip.ec2_eip` as `+ create` instead of refreshing | Resources exist in AWS but are not in `terraform.tfstate` | Run `terraform import` for each resource before applying |
| Missing `terraform init` | Provider not found error during plan or apply | `.terraform/providers` directory absent or corrupt | Run `terraform init` to reinitialize the working directory |

---

## Lessons Learned

**Verify pre-state before any Terraform modification**

Querying AWS CLI for both the instance ID and the EIP allocation ID before touching `main.tf` establishes a confirmed baseline. When working with pre-provisioned infrastructure, assuming Terraform state reflects reality is a risk. Explicit verification prevents silent mismatches.

**Use attribute references to encode dependencies**

Hardcoding resource IDs in configuration files is a common shortcut that breaks immediately when IDs change across environment promotions (dev, staging, production). Attribute references like `aws_instance.ec2.id` cost nothing and eliminate an entire category of configuration errors.

**`terraform fmt` output indicates which files changed**

When `terraform fmt` lists a file name in its output, that file was modified. When it produces no output for a given file, that file was already correctly formatted. Understanding this output helps distinguish between formatting-related and logic-related configuration issues.

**`terraform plan` is a correctness contract, not just a preview**

Reviewing the plan output for `0 to change, 0 to destroy` before applying is a critical production discipline. In this task, the plan confirmed that existing resources would not be modified or recreated, providing high confidence that the apply would be non-disruptive.

**Cross-validate infrastructure state with AWS CLI after apply**

Terraform state is an internal model. AWS is the source of truth. Running `aws ec2 describe-addresses` after apply confirmed that the association exists in the actual AWS control plane, not just in Terraform's local state file. This two-source validation pattern should be standard practice for any Terraform change affecting networking or compute resources.

---

## Reference

| Resource | Link |
|---|---|
| Terraform `aws_eip_association` | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip_association |
| Terraform `aws_eip` | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/eip |
| Terraform `aws_instance` | https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance |
| AWS CLI `describe-addresses` | https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-addresses.html |
| AWS CLI `describe-instances` | https://docs.aws.amazon.com/cli/latest/reference/ec2/describe-instances.html |
| Terraform CLI Commands | https://developer.hashicorp.com/terraform/cli/commands |










<img width="1048" height="528" alt="image" src="https://github.com/user-attachments/assets/56353cc2-d25f-445e-89e5-6748bf325f34" />

<img width="1046" height="672" alt="image" src="https://github.com/user-attachments/assets/e0435f8d-6cba-42e4-b7a2-99180e8fad62" />
<img width="1044" height="740" alt="image" src="https://github.com/user-attachments/assets/66e7758c-df16-4726-97e3-692e78017923" />
<img width="1048" height="659" alt="image" src="https://github.com/user-attachments/assets/0c8fa3c9-a64c-4fc9-9038-286527988421" />
<img width="1047" height="674" alt="image" src="https://github.com/user-attachments/assets/ce85723c-58da-4a40-9cb5-6b09d62cccd4" />
<img width="1049" height="667" alt="image" src="https://github.com/user-attachments/assets/585b4461-28dd-475f-bf5d-777b20098d41" />
<img width="1044" height="562" alt="image" src="https://github.com/user-attachments/assets/a818feb6-54b9-4e95-a903-012e4c85b4b8" />
<img width="1043" height="645" alt="image" src="https://github.com/user-attachments/assets/41f20a80-1ec3-461d-abc4-e3475ac22ccc" />
<img width="1044" height="636" alt="image" src="https://github.com/user-attachments/assets/a9a0271a-b776-4fd8-9a22-c76a55c8e627" />
<img width="1041" height="762" alt="image" src="https://github.com/user-attachments/assets/e548850b-7524-4672-89d9-c842e9315c1b" />
<img width="1048" height="494" alt="image" src="https://github.com/user-attachments/assets/6bf5c826-0bdf-4896-bbc3-6d529ee7b93a" />
<img width="1049" height="551" alt="image" src="https://github.com/user-attachments/assets/386a734e-09ce-4376-8929-3ac241696d5f" />
