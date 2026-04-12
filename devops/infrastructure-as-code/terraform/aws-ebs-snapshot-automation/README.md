# Terraform EBS Volume Snapshot Provisioning on AWS (LocalStack)

> Provisioning an EBS snapshot from an existing volume using Terraform IaC in a LocalStack-emulated AWS environment, following enterprise infrastructure-as-code standards.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Architecture Summary](#architecture-summary)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1: Verify Working Directory](#step-1-verify-working-directory)
  * [Step 2: Inspect Existing Configuration Files](#step-2-inspect-existing-configuration-files)
  * [Step 3: Append EBS Snapshot Resource to main.tf](#step-3-append-ebs-snapshot-resource-to-maintf)
  * [Step 4: Verify the Final main.tf](#step-4-verify-the-final-maintf)
  * [Step 5: Apply the Terraform Configuration](#step-5-apply-the-terraform-configuration)
  * [Step 6: Validate Provisioned Resources](#step-6-validate-provisioned-resources)
* [Resource Specifications](#resource-specifications)
* [Errors and Resolutions](#errors-and-resolutions)
* [Best Practices Applied](#best-practices-applied)
* [Lessons Learned](#lessons-learned)

---

## Project Overview

The Nautilus DevOps team manages EBS volumes across multiple AWS regions and requires automated snapshot capabilities to support regular data backup operations. This task provisions a point-in-time EBS snapshot from an existing volume using Terraform, targeting a LocalStack-emulated AWS environment.

**Objective:** Extend an existing Terraform configuration to create an EBS snapshot (`devops-vol-ss`) from an already-provisioned EBS volume (`devops-vol`) in the `us-east-1` region, using only the existing `main.tf` file without introducing any additional `.tf` files.

**Tooling:**

* Terraform (with AWS provider `5.91.0`)
* LocalStack (AWS emulation endpoint: `http://aws:4566`)
* AWS region: `us-east-1`, availability zone: `us-east-1a`

---

## Architecture Summary

```
LocalStack (http://aws:4566)
        |
        +-- aws_ebs_volume "k8s_volume"
        |       Name: devops-vol
        |       Size: 5 GiB
        |       Type: gp2
        |       AZ:   us-east-1a
        |       ID:   vol-301348c5b2d48b6be
        |
        +-- aws_ebs_snapshot "devops_snapshot"
                Name:        devops-vol-ss
                Description: Devops Snapshot
                Source:      aws_ebs_volume.k8s_volume.id
                ID:          snap-a216ce335cea1e4bf
```

The snapshot resource references the volume using Terraform's implicit dependency resolution via `aws_ebs_volume.k8s_volume.id`, ensuring correct provisioning order without explicit `depends_on` declarations.

---

## Prerequisites

* Terraform CLI installed and initialized (`.terraform/` directory and `.terraform.lock.hcl` already present)
* LocalStack running and reachable at `http://aws:4566`
* Existing `provider.tf` configured with the `hashicorp/aws` provider version `5.91.0` and LocalStack endpoint overrides
* The EBS volume `devops-vol` already provisioned and tracked in `terraform.tfstate`
* Working directory: `/home/bob/terraform`

---

## Repository Structure

```
/home/bob/terraform/
├── .terraform/                  # Provider plugins (pre-initialized)
├── .terraform.lock.hcl          # Provider dependency lock file
├── main.tf                      # Core resource definitions (EBS volume + snapshot)
├── provider.tf                  # AWS provider and LocalStack endpoint configuration
├── terraform.tfstate            # Terraform state file
└── README.MD                    # Original task readme
```

---

## Implementation Guide

### Step 1: Verify Working Directory

Confirm the Terraform working directory and inspect all present files before making any changes.

```bash
pwd
ls -la
```

**Output:**

```
/home/bob/terraform

total 40
drwxr-xr-x 1 bob bob 4096 Apr 12 00:52 .
drwxr-x--- 1 bob bob 4096 Apr 12 00:52 ..
drwxr-xr-x 3 bob bob 4096 Apr 12 00:52 .terraform
-rw-r--r-- 1 bob bob 1406 Apr 12 00:52 .terraform.lock.hcl
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-r--r-- 1 bob bob  176 Apr 12 00:52 main.tf
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
-rw-rw-r-- 1 bob bob 1324 Apr 12 00:52 terraform.tfstate
```

> Screenshot: Directory listing confirming all expected Terraform files are present in `/home/bob/terraform`

<img width="1039" height="520" alt="image" src="https://github.com/user-attachments/assets/6543b1b6-99d4-41e4-b848-191937b91349" />

---

### Step 2: Inspect Existing Configuration Files

Review the existing `main.tf` and `provider.tf` to understand the current resource state and provider configuration before introducing changes.

**Inspect main.tf:**

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_ebs_volume" "k8s_volume" {
  availability_zone = "us-east-1a"
  size              = 5
  type              = "gp2"

  tags = {
    Name = "devops-vol"
  }
}
```

**Inspect provider.tf:**

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

> Screenshot: Terminal output showing the contents of `main.tf` and `provider.tf` as inspected before modification

---

### Step 3: Append EBS Snapshot Resource to main.tf

Per the task constraint, the snapshot resource must be added directly to `main.tf` without creating any new `.tf` files. A heredoc append is used to safely add the resource block.

```bash
cat >> main.tf << 'EOF'

resource "aws_ebs_snapshot" "devops_snapshot" {
  volume_id   = aws_ebs_volume.k8s_volume.id
  description = "Devops Snapshot"

  tags = {
    Name = "devops-vol-ss"
  }
}
EOF
```

**Key design decisions:**

* `volume_id = aws_ebs_volume.k8s_volume.id` creates an implicit dependency, ensuring the volume is resolved before the snapshot is created.
* `description = "Devops Snapshot"` matches the exact case specified in the task requirements.
* The `Name` tag is set to `devops-vol-ss` per the task specification.

> Screenshot: Terminal showing the heredoc append command executed successfully with no errors

---

### Step 4: Verify the Final main.tf

Confirm the complete and correct state of `main.tf` after appending the snapshot resource.

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_ebs_volume" "k8s_volume" {
  availability_zone = "us-east-1a"
  size              = 5
  type              = "gp2"

  tags = {
    Name = "devops-vol"
  }
}

resource "aws_ebs_snapshot" "devops_snapshot" {
  volume_id   = aws_ebs_volume.k8s_volume.id
  description = "Devops Snapshot"

  tags = {
    Name = "devops-vol-ss"
  }
}
```

> Screenshot: Full `main.tf` content confirming both the EBS volume and EBS snapshot resource blocks are present and correctly structured

---

### Step 5: Apply the Terraform Configuration

Execute `terraform apply` with the `--auto-approve` flag to provision the snapshot resource without interactive confirmation. The existing volume is refreshed from state, and only the new snapshot resource is created.

```bash
terraform apply --auto-approve
```

**Output:**

```
aws_ebs_volume.k8s_volume: Refreshing state... [id=vol-301348c5b2d48b6be]

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the
following symbols:
  + create

Terraform will perform the following actions:

  # aws_ebs_snapshot.devops_snapshot will be created
  + resource "aws_ebs_snapshot" "devops_snapshot" {
      + arn                    = (known after apply)
      + data_encryption_key_id = (known after apply)
      + description            = "Devops Snapshot"
      + encrypted              = (known after apply)
      + id                     = (known after apply)
      + kms_key_id             = (known after apply)
      + owner_alias            = (known after apply)
      + owner_id               = (known after apply)
      + storage_tier           = (known after apply)
      + tags                   = {
          + "Name" = "devops-vol-ss"
        }
      + tags_all               = {
          + "Name" = "devops-vol-ss"
        }
      + volume_id              = "vol-301348c5b2d48b6be"
      + volume_size            = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
aws_ebs_snapshot.devops_snapshot: Creating...
aws_ebs_snapshot.devops_snapshot: Creation complete after 0s [id=snap-a216ce335cea1e4bf]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Apply summary:**

| Metric | Value |
|---|---|
| Resources added | 1 |
| Resources changed | 0 |
| Resources destroyed | 0 |
| Snapshot ID | `snap-a216ce335cea1e4bf` |
| Source volume ID | `vol-301348c5b2d48b6be` |

> Screenshot: Full `terraform apply` output including the execution plan and successful creation confirmation

---

### Step 6: Validate Provisioned Resources

Run `terraform show` to confirm the complete state of both the EBS volume and the newly created EBS snapshot, validating all required attributes.

```bash
terraform show
```

**Output:**

```
# aws_ebs_snapshot.devops_snapshot:
resource "aws_ebs_snapshot" "devops_snapshot" {
    arn                    = "arn:aws:ec2:us-east-1::snapshot/snap-a216ce335cea1e4bf"
    data_encryption_key_id = null
    description            = "Devops Snapshot"
    encrypted              = false
    id                     = "snap-a216ce335cea1e4bf"
    kms_key_id             = null
    outpost_arn            = null
    owner_alias            = null
    owner_id               = "000000000000"
    storage_tier           = null
    tags                   = {
        "Name" = "devops-vol-ss"
    }
    tags_all               = {
        "Name" = "devops-vol-ss"
    }
    volume_id              = "vol-301348c5b2d48b6be"
    volume_size            = 5
}

# aws_ebs_volume.k8s_volume:
resource "aws_ebs_volume" "k8s_volume" {
    arn                  = "arn:aws:ec2:us-east-1::volume/vol-301348c5b2d48b6be"
    availability_zone    = "us-east-1a"
    encrypted            = false
    final_snapshot       = false
    id                   = "vol-301348c5b2d48b6be"
    iops                 = 0
    kms_key_id           = null
    multi_attach_enabled = false
    outpost_arn          = null
    size                 = 5
    snapshot_id          = null
    tags                 = {
        "Name" = "devops-vol"
    }
    tags_all             = {
        "Name" = "devops-vol"
    }
    throughput           = 0
    type                 = "gp2"
}
```

**Validation checklist:**

| Requirement | Expected | Actual | Status |
|---|---|---|---|
| Snapshot Name tag | `devops-vol-ss` | `devops-vol-ss` | PASS |
| Snapshot description | `Devops Snapshot` | `Devops Snapshot` | PASS |
| Source volume ID | `vol-301348c5b2d48b6be` | `vol-301348c5b2d48b6be` | PASS |
| Snapshot volume size | `5 GiB` | `5` | PASS |
| Snapshot region | `us-east-1` | ARN confirms `us-east-1` | PASS |
| Snapshot ID assigned | Non-null | `snap-a216ce335cea1e4bf` | PASS |

> Screenshot: Full `terraform show` output confirming both `aws_ebs_snapshot.devops_snapshot` and `aws_ebs_volume.k8s_volume` in state with all expected attributes

---

## Resource Specifications

### aws_ebs_volume.k8s_volume

| Attribute | Value |
|---|---|
| Resource label | `k8s_volume` |
| Name tag | `devops-vol` |
| Availability zone | `us-east-1a` |
| Size | `5 GiB` |
| Type | `gp2` |
| Assigned ID | `vol-301348c5b2d48b6be` |

### aws_ebs_snapshot.devops_snapshot

| Attribute | Value |
|---|---|
| Resource label | `devops_snapshot` |
| Name tag | `devops-vol-ss` |
| Description | `Devops Snapshot` |
| Source volume | `aws_ebs_volume.k8s_volume.id` |
| Resolved volume ID | `vol-301348c5b2d48b6be` |
| Assigned snapshot ID | `snap-a216ce335cea1e4bf` |
| Volume size captured | `5 GiB` |
| ARN | `arn:aws:ec2:us-east-1::snapshot/snap-a216ce335cea1e4bf` |

---

## Errors and Resolutions

No errors were encountered during this implementation. The Terraform plan resolved cleanly on the first apply with a single resource addition and zero changes or destructions to existing infrastructure.

**Potential failure scenarios and their resolutions:**

| Scenario | Cause | Resolution |
|---|---|---|
| `volume_id` not found | Volume not in state or wrong resource reference | Verify volume resource name matches reference: `aws_ebs_volume.<label>.id` |
| LocalStack connection refused | LocalStack container not running or incorrect endpoint | Confirm `http://aws:4566` is reachable; check `provider.tf` endpoint block |
| State drift on volume | Volume manually deleted outside Terraform | Run `terraform refresh` or `terraform import` to reconcile state |
| Snapshot creation timeout | LocalStack resource contention | Retry `terraform apply`; LocalStack snapshots are typically instant |

---

## Best Practices Applied

* **Implicit dependency over explicit:** Used `aws_ebs_volume.k8s_volume.id` as the `volume_id` reference rather than a hardcoded volume ID string. This ensures Terraform's dependency graph correctly orders resource creation and avoids orphaned snapshot attempts.

* **Single file constraint respected:** The snapshot resource was appended directly to the existing `main.tf` using a heredoc, in strict adherence to the task requirement of not creating additional `.tf` files.

* **Heredoc for safe file mutation:** Used `cat >> main.tf << 'EOF' ... EOF` to append content safely without overwriting existing resource definitions, reducing risk of configuration loss.

* **Pre-apply file verification:** Ran `cat main.tf` after the append to visually confirm the final configuration before executing `terraform apply`, catching any formatting or truncation issues early.

* **State inspection post-apply:** Used `terraform show` rather than relying solely on apply output to confirm the complete resource state, including all computed attributes such as `volume_size`, `arn`, and `owner_id`.

* **Tag consistency:** Applied the `Name` tag using the same key casing pattern (`Name`) as the source volume resource, maintaining uniform tagging conventions across all managed resources.

* **LocalStack provider hardening:** The `provider.tf` includes `skip_credentials_validation = true` and `skip_requesting_account_id = true`, which are required and correct for LocalStack environments to prevent failed AWS credential handshakes.

---

## Lessons Learned

* **Implicit references enforce dependency ordering at no cost.** Referencing `aws_ebs_volume.k8s_volume.id` instead of a static volume ID string costs nothing in configuration complexity but guarantees Terraform will always create the volume before the snapshot. This pattern is essential in any resource chain where one resource depends on the runtime-assigned ID of another.

* **State refresh prevents unnecessary recreation.** On apply, Terraform refreshed the existing volume from state (`Refreshing state... [id=vol-301348c5b2d48b6be]`) before planning. This confirms that Terraform correctly identified the volume as already provisioned, added only the snapshot to the plan, and left the volume unchanged. Understanding this behavior is critical for safely extending existing infrastructure configurations.

* **LocalStack snapshot completion is near-instant but production is not.** In LocalStack, snapshot creation completed in `0s`. In production AWS environments, EBS snapshots can take minutes to hours depending on volume size and I/O activity. Pipelines that depend on snapshot completion status should use a waiter or poll the snapshot `state` attribute (`completed`) before proceeding.

* **Heredoc single-quote quoting prevents variable expansion.** The `'EOF'` quoting pattern in `cat >> main.tf << 'EOF'` is intentional. Single-quoting the delimiter prevents the shell from interpreting `$` characters or backticks inside the block, which is essential when the content includes Terraform interpolation syntax or any string that resembles a shell variable.

* **`terraform show` is more complete than apply output for validation.** The apply output confirms creation but does not display all computed attributes. `terraform show` surfaces the full resource schema in its final state, including ARN, owner ID, volume size, and encryption status, making it the correct tool for post-deployment validation and documentation.




<img width="1045" height="627" alt="image" src="https://github.com/user-attachments/assets/73c06170-eaf2-410a-bf3f-a14640818280" />
<img width="1069" height="736" alt="image" src="https://github.com/user-attachments/assets/d3299ea7-aebb-472e-b0ed-1912325476ee" />
<img width="1034" height="695" alt="image" src="https://github.com/user-attachments/assets/b26513ab-0576-4d25-a63a-fb7aa97ebd0d" />
<img width="1047" height="734" alt="image" src="https://github.com/user-attachments/assets/34bbef91-1b9d-430d-a040-fd26fe8da573" />
<img width="1046" height="732" alt="image" src="https://github.com/user-attachments/assets/47f18c11-d1ea-49d6-b07f-1736082cbec4" />
<img width="1074" height="696" alt="image" src="https://github.com/user-attachments/assets/26cd7a00-bd4d-410f-9637-e3fb06573b8a" />
