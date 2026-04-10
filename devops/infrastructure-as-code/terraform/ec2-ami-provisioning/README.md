# Terraform: Create AMI from Existing EC2 Instance (AWS LocalStack)

> **Project:** Nautilus Infrastructure Migration | AWS Cloud IaC Automation
> **Engineer Role:** DevOps / Infrastructure Engineer
> **Toolchain:** Terraform 1.11.0 | AWS Provider 5.91.0 | LocalStack (aws:4566)
> **Outcome:** AMI `nautilus-ec2-ami` successfully created from EC2 instance `nautilus-ec2` and confirmed in `available` state.

---

## Table of Contents

1. [Project Context](#1-project-context)
2. [Problem Statement](#2-problem-statement)
3. [Solution Architecture](#3-solution-architecture)
4. [Prerequisites](#4-prerequisites)
5. [Implementation Guide](#6-implementation-guide)
   - [Step 1: Inspect the Working Directory](#step-1-inspect-the-working-directory)
   - [Step 2: Review Existing Terraform Configuration](#step-2-review-existing-terraform-configuration)
   - [Step 3: Review the Provider Configuration](#step-3-review-the-provider-configuration)
   - [Step 4: Review the Existing State File](#step-4-review-the-existing-state-file)
   - [Step 5: Append the AMI Resource to main.tf](#step-5-append-the-ami-resource-to-maintf)
   - [Step 6: Verify the Updated main.tf](#step-6-verify-the-updated-maintf)
   - [Step 7: Validate the Terraform Configuration](#step-7-validate-the-terraform-configuration)
   - [Step 8: Generate the Execution Plan](#step-8-generate-the-execution-plan)
   - [Step 9: Apply the Configuration](#step-9-apply-the-configuration)
   - [Step 10: Verify the AMI via AWS CLI](#step-10-verify-the-ami-via-aws-cli)
6. [Key Resource Attributes](#7-key-resource-attributes)
7. [Best Practices Applied](#8-best-practices-applied)
8. [Lessons Learned](#9-lessons-learned)
9. [Troubleshooting Reference](#10-troubleshooting-reference)

---

## 1. Project Context

The **Nautilus DevOps team** is executing a phased migration of on-premises infrastructure to AWS. To reduce risk, the migration is broken into discrete, independently verifiable units. This task represents one such unit: capturing a golden image (AMI) of a running EC2 instance using Terraform, enabling the team to replicate the instance state reliably across environments during subsequent migration phases.

All infrastructure in this phase is provisioned against a **LocalStack** environment simulating AWS services at `http://aws:4566`, allowing safe iteration before targeting production AWS accounts.

---

## 2. Problem Statement

Given an existing, running EC2 instance named `nautilus-ec2` (already tracked in Terraform state), the requirement is to:

- Create an AMI named `nautilus-ec2-ami` derived from the running instance
- Manage the AMI resource entirely within the existing `main.tf` file (no additional `.tf` files)
- Confirm the AMI reaches `available` state post-creation
- Maintain a clean, valid Terraform configuration throughout

---

## 3. Solution Architecture

```
Existing State                        New Resource
+--------------------+                +-------------------------------+
|  aws_instance.ec2  |   source_id    |  aws_ami_from_instance        |
|  id: i-790313c4.. | ------------> |  name: nautilus-ec2-ami       |
|  name: nautilus-ec2|                |  id: ami-9c1d2a2f8e2bd8358    |
|  state: running    |                |  state: available             |
+--------------------+                +-------------------------------+
         |                                         |
         +------------- LocalStack ----------------+
                       http://aws:4566
```

The `aws_ami_from_instance` resource references the `aws_instance.ec2` resource directly via `aws_instance.ec2.id`, ensuring the AMI always reflects the correct source instance ID as tracked in state, rather than a hardcoded value.

---

## 4. Prerequisites

| Requirement | Version / Detail |
|---|---|
| Terraform | 1.11.0 |
| AWS Terraform Provider | hashicorp/aws 5.91.0 |
| LocalStack | Running and reachable at `http://aws:4566` |
| AWS CLI | Configured with LocalStack endpoint |
| Working Directory | `/home/bob/terraform` |
| Existing EC2 Instance | `nautilus-ec2` already provisioned and tracked in state |

---

## 5. Implementation Guide

### Step 1: Inspect the Working Directory

Open an integrated terminal from VS Code (right-click the `EXPLORER` panel and select **Open in Integrated Terminal**) and confirm all expected files are present.

```bash
ls -la
```

**Expected output:**

```
total 44
drwxr-xr-x 1 bob bob 4096 Apr 10 16:05 .
drwxr-x--- 1 bob bob 4096 Apr 10 16:05 ..
drwxr-xr-x 3 bob bob 4096 Apr 10 16:05 .terraform
-rw-r--r-- 1 bob bob 1406 Apr 10 16:05 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  231 Apr 10 16:05 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-r--r-- 1 bob bob 4097 Apr 10 16:05 terraform.tfstate
```

> **Screenshot: `ls -la` output confirming directory contents**

<img width="1100" height="489" alt="image" src="https://github.com/user-attachments/assets/ef54ba00-7edd-4849-a029-2760c4a7bdc0" />

---

### Step 2: Review Existing Terraform Configuration

Inspect `main.tf` to understand the existing resource and confirm the instance name and tags before modifying.

```bash
cat main.tf
```

**Existing content:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  vpc_security_group_ids = [
    "sg-97096b9d2ffa12df1"
  ]

  tags = {
    Name = "nautilus-ec2"
  }
}
```

> **Screenshot: `cat main.tf` showing the existing EC2 resource block**

<img width="1095" height="564" alt="image" src="https://github.com/user-attachments/assets/e883a733-21b1-4fc2-b9e9-9facd5bc621f" />

Key observations:
- The instance resource label is `ec2` under the `aws_instance` type.
- The instance ID (`i-790313c4ff594075c`) is already recorded in state and will be referenced dynamically in the next step.

---

### Step 3: Review the Provider Configuration

Inspect `provider.tf` to confirm the LocalStack endpoint mapping and AWS provider version constraints.

```bash
cat provider.tf
```

**Content:**

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

> **Screenshot: `cat provider.tf` confirming LocalStack endpoints**

<img width="1129" height="778" alt="image" src="https://github.com/user-attachments/assets/acaec1c8-b19d-4747-95da-d1acb2a961e4" />

Notable configuration flags:
- `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack compatibility, as it does not enforce real AWS credential chains.
- All service endpoints are uniformly directed to `http://aws:4566`.

---

### Step 4: Review the Existing State File

Inspect `terraform.tfstate` to confirm the existing EC2 instance ID and verify the resource is actively tracked.

```bash
cat terraform.tfstate
```

Key values extracted from state:

| Attribute | Value |
|---|---|
| Instance ID | `i-790313c4ff594075c` |
| Instance Name | `nautilus-ec2` |
| Instance State | `running` |
| Private IP | `10.85.111.44` |
| Public IP | `54.214.246.10` |
| AMI | `ami-0c101f26f147fa7fd` |
| Instance Type | `t2.micro` |

> **Screenshot: `cat terraform.tfstate` showing the tracked EC2 instance attributes**

This confirms the source instance is running and tracked, making it a valid target for AMI creation.

---

### Step 5: Append the AMI Resource to main.tf

Append the `aws_ami_from_instance` resource block to `main.tf` using a heredoc. This approach avoids creating a separate `.tf` file, satisfying the task constraint.

```bash
cat >> main.tf << 'EOF'

# Create AMI from existing EC2 instance
resource "aws_ami_from_instance" "nautilus_ec2_ami" {
  name               = "nautilus-ec2-ami"
  source_instance_id = aws_instance.ec2.id

  tags = {
    Name = "nautilus-ec2-ami"
  }
}
EOF
```

> **Screenshot: Terminal showing successful heredoc append to main.tf**

**Design decision:** `source_instance_id = aws_instance.ec2.id` uses an implicit resource reference rather than a hardcoded instance ID. This ensures the AMI creation is always tied to the correct resource as tracked in state, and correctly models the dependency in the Terraform DAG.

---

### Step 6: Verify the Updated main.tf

Confirm the file was updated correctly and both resource blocks are present.

```bash
cat main.tf
```

**Expected content after update:**

```hcl
# Provision EC2 instance
resource "aws_instance" "ec2" {
  ami           = "ami-0c101f26f147fa7fd"
  instance_type = "t2.micro"
  vpc_security_group_ids = [
    "sg-97096b9d2ffa12df1"
  ]

  tags = {
    Name = "nautilus-ec2"
  }
}
# Create AMI from existing EC2 instance
resource "aws_ami_from_instance" "nautilus_ec2_ami" {
  name               = "nautilus-ec2-ami"
  source_instance_id = aws_instance.ec2.id

  tags = {
    Name = "nautilus-ec2-ami"
  }
}
```

> **Screenshot: `cat main.tf` showing both resource blocks present**

---

### Step 7: Validate the Terraform Configuration

Run `terraform validate` to confirm there are no syntax or schema errors in the configuration before planning.

```bash
terraform validate
```

**Expected output:**

```
Success! The configuration is valid.
```

> **Screenshot: `terraform validate` returning success**

---

### Step 8: Generate the Execution Plan

Run `terraform plan` to preview the changes Terraform will apply. This is a non-destructive, read-only operation.

```bash
terraform plan
```

**Expected output (abridged):**

```
aws_instance.ec2: Refreshing state... [id=i-790313c4ff594075c]

Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ami_from_instance.nautilus_ec2_ami will be created
  + resource "aws_ami_from_instance" "nautilus_ec2_ami" {
      + name               = "nautilus-ec2-ami"
      + source_instance_id = "i-790313c4ff594075c"
      + tags               = {
          + "Name" = "nautilus-ec2-ami"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> **Screenshot: `terraform plan` output showing the `+create` action for `aws_ami_from_instance.nautilus_ec2_ami`**

Key verification points from the plan:
- `source_instance_id = "i-790313c4ff594075c"` confirms the correct source instance is resolved from state.
- `Plan: 1 to add, 0 to change, 0 to destroy` confirms no unintended modifications to the existing EC2 instance.

---

### Step 9: Apply the Configuration

Apply the plan with `-auto-approve` to bypass the interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Expected output (abridged):**

```
aws_instance.ec2: Refreshing state... [id=i-790313c4ff594075c]

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ami_from_instance.nautilus_ec2_ami: Creating...
aws_ami_from_instance.nautilus_ec2_ami: Creation complete after 5s [id=ami-9c1d2a2f8e2bd8358]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

> **Screenshot: `terraform apply -auto-approve` showing creation complete with AMI ID `ami-9c1d2a2f8e2bd8358`**

The AMI was created in approximately 5 seconds within the LocalStack environment.

---

### Step 10: Verify the AMI via AWS CLI

Use the AWS CLI with the LocalStack endpoint to confirm the AMI exists and is in `available` state.

```bash
aws ec2 describe-images \
  --endpoint-url http://aws:4566 \
  --filters "Name=name,Values=nautilus-ec2-ami" \
  --query "Images[*].{ID:ImageId,Name:Name,State:State}" \
  --output table
```

**Expected output:**

```
------------------------------------------------------------
|                      DescribeImages                      |
+------------------------+--------------------+------------+
|           ID           |       Name         |   State    |
+------------------------+--------------------+------------+
|  ami-9c1d2a2f8e2bd8358 |  nautilus-ec2-ami  |  available |
+------------------------+--------------------+------------+
```

> **Screenshot: AWS CLI `describe-images` output confirming AMI `ami-9c1d2a2f8e2bd8358` is `available`**

All three required conditions are confirmed:
- AMI ID: `ami-9c1d2a2f8e2bd8358`
- Name: `nautilus-ec2-ami`
- State: `available`

---

## 6. Key Resource Attributes

### aws_instance.ec2 (Pre-existing)

| Attribute | Value |
|---|---|
| Resource Label | `ec2` |
| AMI | `ami-0c101f26f147fa7fd` |
| Instance Type | `t2.micro` |
| Instance ID | `i-790313c4ff594075c` |
| Instance State | `running` |
| Tag: Name | `nautilus-ec2` |

### aws_ami_from_instance.nautilus_ec2_ami (Created)

| Attribute | Value |
|---|---|
| Resource Label | `nautilus_ec2_ami` |
| AMI Name | `nautilus-ec2-ami` |
| AMI ID | `ami-9c1d2a2f8e2bd8358` |
| Source Instance ID | `i-790313c4ff594075c` |
| State | `available` |
| Tag: Name | `nautilus-ec2-ami` |

---

## 7. Best Practices Applied

**Resource referencing over hardcoding:** The `source_instance_id` attribute uses `aws_instance.ec2.id` rather than a literal instance ID string. This builds an explicit dependency edge in the Terraform resource graph and ensures idempotency across environments where instance IDs may differ.

**Single file constraint respected:** The AMI resource was appended to the existing `main.tf` using a heredoc rather than creating a new `.tf` file. While splitting resources across files is a common pattern for larger codebases, respecting the single-file constraint here reflects the principle of minimal footprint for small, scoped tasks.

**Validate before plan, plan before apply:** The three-step sequence (`validate` -> `plan` -> `apply`) was followed strictly. `terraform validate` catches configuration errors early without requiring provider communication. `terraform plan` provides a human-readable diff for review before any state mutation occurs.

**Tag consistency:** Both the EC2 instance and the AMI use the `Name` tag pattern aligned with the resource name, which is a standard AWS resource tagging convention and aids in cost attribution, filtering, and team visibility.

**LocalStack endpoint isolation:** The provider configuration routes all AWS API calls to LocalStack, ensuring zero risk of accidental interaction with production AWS accounts during development and testing phases.

---

## 8. Lessons Learned

**State review before resource creation is critical.** Reviewing `terraform.tfstate` prior to writing new resources confirmed the instance ID and operational state without requiring an additional AWS CLI call. In production environments, always verify that source resources are in the expected state before referencing them in dependent resource definitions.

**Heredoc append is safe for single-file modifications.** Using `cat >> main.tf << 'EOF'` is a reliable, auditable way to append content to an existing Terraform file in a terminal-only environment. The single-quoted `'EOF'` delimiter prevents shell variable expansion inside the heredoc, which is essential when the content includes Terraform interpolations such as `aws_instance.ec2.id`.

**`-auto-approve` is appropriate in controlled lab environments only.** In production pipelines, always require explicit plan review and approval gates. The `-auto-approve` flag was used here because the plan output had already been reviewed in the preceding step and the environment is isolated LocalStack.

**LocalStack AMI creation is near-instantaneous.** Real AWS AMI creation from a running instance can take several minutes and may require the instance to be stopped first depending on configuration. The 5-second creation time observed here is specific to LocalStack's mock implementation and should not be used as a benchmark for production timelines.

---

## 9. Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `terraform validate` fails with unknown resource type | `aws_ami_from_instance` not supported by provider version | Confirm provider version is 5.x or later; run `terraform init -upgrade` |
| `Error: Instance not found` during apply | EC2 instance ID in state is stale or instance was deleted outside Terraform | Run `terraform refresh` to sync state, then re-plan |
| AMI state shows `pending` indefinitely | LocalStack version issue or EC2 service not running | Restart LocalStack; verify `http://aws:4566` is reachable |
| `describe-images` returns empty results | Filter mismatch or wrong endpoint | Confirm `--endpoint-url http://aws:4566` is set; verify AMI name matches exactly |
| `Error: No valid credential sources found` | Credentials not configured for LocalStack | Ensure `skip_credentials_validation = true` is set in provider block; export dummy `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` env vars if required |

---







<img width="1138" height="755" alt="image" src="https://github.com/user-attachments/assets/bd0b46e1-dc63-47a1-823b-735d98e3f58c" />
<img width="1139" height="773" alt="image" src="https://github.com/user-attachments/assets/b744a8c0-a9f0-4982-b178-21c43eb42edf" />
<img width="1147" height="773" alt="image" src="https://github.com/user-attachments/assets/b5b7b3ff-6288-4946-a701-bd2d4a5895e8" />
<img width="1139" height="729" alt="image" src="https://github.com/user-attachments/assets/d51f8576-cde3-46ab-92dc-679c28ca1e82" />
<img width="1136" height="727" alt="image" src="https://github.com/user-attachments/assets/f53a4bfb-86ec-4ce9-a672-f6bc9a5d188f" />
<img width="1173" height="800" alt="image" src="https://github.com/user-attachments/assets/3d287026-2992-415d-a142-a31bae611248" />
<img width="1172" height="815" alt="image" src="https://github.com/user-attachments/assets/d53c3c44-bb40-496c-8837-a6ee434ab4be" />
<img width="1153" height="819" alt="image" src="https://github.com/user-attachments/assets/2cc97347-84c6-4d2e-a644-5294b90f75fb" />
<img width="1134" height="734" alt="image" src="https://github.com/user-attachments/assets/d35edca2-e3e1-4e85-9a44-f75b32294de6" />


