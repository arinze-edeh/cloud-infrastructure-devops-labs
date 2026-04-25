# Terraform AWS OpenSearch Domain Provisioning via LocalStack

> Provisioning a production-style Amazon OpenSearch Service domain using Terraform IaC against a LocalStack-emulated AWS environment for the Nautilus DevOps infrastructure platform.

---

## Table of Contents

* [Project Overview](#project-overview)
* [Problem Statement](#problem-statement)
* [Solution Architecture](#solution-architecture)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [Implementation Guide](#implementation-guide)
  * [Step 1 - Verify Working Directory and Existing Configuration](#step-1---verify-working-directory-and-existing-configuration)
  * [Step 2 - Inspect the Provider Configuration](#step-2---inspect-the-provider-configuration)
  * [Step 3 - Author the Main Terraform Configuration](#step-3---author-the-main-terraform-configuration)
  * [Step 4 - Verify the Configuration File](#step-4---verify-the-configuration-file)
  * [Step 5 - Initialize the Terraform Working Directory](#step-5---initialize-the-terraform-working-directory)
  * [Step 6 - Patch the Instance Type Value](#step-6---patch-the-instance-type-value)
  * [Step 7 - Validate the Configuration](#step-7---validate-the-configuration)
  * [Step 8 - Apply the Infrastructure](#step-8---apply-the-infrastructure)
  * [Step 9 - Confirm Infrastructure Convergence](#step-9---confirm-infrastructure-convergence)
  * [Step 10 - Verify the Domain via AWS CLI](#step-10---verify-the-domain-via-aws-cli)
  * [Step 11 - Confirm Terraform State](#step-11---confirm-terraform-state)
* [Resource Specification](#resource-specification)
* [Validation Results](#validation-results)
* [Best Practices Applied](#best-practices-applied)
* [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
* [Lessons Learned](#lessons-learned)
* [Technologies Used](#technologies-used)

---

## Project Overview

The Nautilus DevOps team required a scalable, searchable log storage backend to support application observability requirements. This project provisions an **Amazon OpenSearch Service domain** named `nautilus-es` using **Terraform** as the sole Infrastructure-as-Code tool, executed against a **LocalStack**-emulated AWS environment running on the `iac-server`.

The entire provisioning lifecycle, from directory inspection through domain health verification, is codified exclusively within a single `main.tf` file alongside the pre-existing `provider.tf`, adhering to strict IaC immutability principles.

---

## Problem Statement

The Nautilus platform lacked a centralized, queryable log store capable of ingesting and indexing application-generated events at scale. The team needed an OpenSearch domain with:

* A deterministic, team-wide domain name (`nautilus-es`)
* A defined engine version (`OpenSearch_1.3`) for compatibility with existing log shippers
* Controlled compute and storage footprint appropriate for a single-node search cluster
* An open access policy permitting internal service-to-service log ingestion without per-request signing overhead
* Full Infrastructure-as-Code traceability so the domain can be reproduced, version-controlled, and audited

The entire provisioning workflow had to target a **LocalStack** endpoint to enable environment parity testing without incurring real AWS costs.

---

## Solution Architecture

```
┌────────────────────────────────────────────────────────────┐
│                      iac-server (bob)                      │
│                                                            │
│   ~/terraform/                                             │
│   ├── provider.tf      (pre-existing, LocalStack config)   │
│   └── main.tf          (authored in this implementation)   │
│                                                            │
│   Terraform CLI  ──► LocalStack (http://aws:4566)          │
│                            │                               │
│                            ▼                               │
│              aws_opensearch_domain.nautilus_es             │
│              domain-name : nautilus-es                     │
│              engine      : OpenSearch_1.3                  │
│              instance    : t3.small.search (x1)            │
│              storage     : 10 GiB gp2 EBS                  │
└────────────────────────────────────────────────────────────┘
```

All AWS API calls are transparently redirected to `http://aws:4566` via the endpoint overrides declared in `provider.tf`. No real AWS credentials or billable resources are involved.

---

## Prerequisites

| Requirement | Details |
|---|---|
| Terraform CLI | v1.x (hashicorp/aws provider 5.91.0 resolved at init) |
| AWS CLI | Configured or available for endpoint-targeted verification |
| LocalStack | Running and accessible at `http://aws:4566` |
| OS User | `bob` on `iac-server` |
| Working directory | `/home/bob/terraform` |

---

## Repository Structure

```
~/terraform/
├── provider.tf          # AWS provider pinned to 5.91.0, LocalStack endpoint map
├── main.tf              # OpenSearch domain resource definition (authored here)
├── README.MD            # Pre-existing task reference document
└── .terraform/          # Generated by terraform init (not committed)
    └── .terraform.lock.hcl
```

> Only `provider.tf` and `main.tf` are committed artifacts. The `.terraform/` directory and lock file are generated at runtime.

---

## Implementation Guide

### Step 1 - Verify Working Directory and Existing Configuration

Before authoring any new configuration, the working directory was inspected to confirm pre-existing files and the absence of a `main.tf`.

```bash
ls -la
```

**Output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 25 21:52 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

```bash
cat main.tf 2>/dev/null || echo "main.tf does not exist yet"
```

**Output:**

```
main.tf does not exist yet
```

*Screenshot: Working directory listing confirming absence of main.tf*

<img width="1053" height="399" alt="image" src="https://github.com/user-attachments/assets/16501c9d-12ed-45f3-bbcc-82117186d20a" />

This confirmed that `main.tf` was the only file requiring creation and that no prior partial state existed in the directory.

---

### Step 2 - Inspect the Provider Configuration

The pre-existing `provider.tf` was reviewed to understand the LocalStack endpoint mapping, the AWS provider version constraint, and the regional configuration before authoring any resource blocks.

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

*Screenshot: provider.tf contents confirming LocalStack endpoint overrides and provider version*

<img width="1074" height="801" alt="image" src="https://github.com/user-attachments/assets/869db26e-f89b-4c78-b3b3-0e8123a0c075" />

Key observations noted from this review:

* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack compatibility
* The `es` endpoint (used by the OpenSearch/Elasticsearch resource) maps to `http://aws:4566`
* The provider version is pinned to `5.91.0`, which supports the `aws_opensearch_domain` resource type

---

### Step 3 - Author the Main Terraform Configuration

The `main.tf` file was created using a heredoc redirect to ensure exact content fidelity without editor interference. The resource block defines an OpenSearch domain with a single-node cluster, gp2 EBS storage, and a fully open access policy scoped to all `es:*` actions.

```bash
cat > ~/terraform/main.tf << 'EOF'
resource "aws_opensearch_domain" "nautilus_es" {
  domain_name    = "nautilus-es"
  engine_version = "OpenSearch_1.3"

  cluster_config {
    instance_type          = "t3.small.search"
    instance_count         = 1
    zone_awareness_enabled = false
  }

  ebs_options {
    ebs_enabled = true
    volume_type = "gp2"
    volume_size = 10
  }

  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "es:*"
        Resource  = "*"
      }
    ]
  })
}
EOF
```

*Screenshot: heredoc command creating main.tf with the full resource block*

<img width="1048" height="745" alt="image" src="https://github.com/user-attachments/assets/18116b31-95cd-4e35-8d3a-6ff14fbb9c79" />

**Design decisions embedded in this configuration:**

| Parameter | Value | Rationale |
|---|---|---|
| `domain_name` | `nautilus-es` | Required by task specification; identifies the domain within the Nautilus platform |
| `engine_version` | `OpenSearch_1.3` | Stable minor version; compatible with common log shipper clients |
| `instance_type` | `t3.small.search` | Minimal footprint suitable for a single-node development/test cluster |
| `instance_count` | `1` | Single-node; `zone_awareness_enabled` set to `false` accordingly |
| `volume_type` | `gp2` | General purpose SSD; adequate IOPS for a 10 GiB log store |
| `volume_size` | `10` | Minimum viable storage for log indexing in a non-production environment |
| `access_policies` | `es:* / Principal: * / Effect: Allow` | Permissive internal access policy; not intended for internet-facing workloads |

---

### Step 4 - Verify the Configuration File

The newly created file was read back to confirm the heredoc wrote the intended content without truncation or encoding issues.

```bash
cat main.tf
```

**Output:**

```hcl
resource "aws_opensearch_domain" "nautilus_es" {
  domain_name    = "nautilus-es"
  engine_version = "OpenSearch_1.3"

  cluster_config {
    instance_type          = "t3.small.search"
    instance_count         = 1
    zone_awareness_enabled = false
  }

  ebs_options {
    ebs_enabled = true
    volume_type = "gp2"
    volume_size = 10
  }

  access_policies = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "es:*"
        Resource  = "*"
      }
    ]
  })
}
```

*Screenshot: cat main.tf confirming correct heredoc output*

<img width="1059" height="808" alt="image" src="https://github.com/user-attachments/assets/2d752df3-1f71-423c-92ab-4f02a880330f" />

---

### Step 5 - Initialize the Terraform Working Directory

`terraform init` was executed to download the pinned `hashicorp/aws` provider at version `5.91.0` and generate the dependency lock file.

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
```

*Screenshot: terraform init output confirming provider installation and lock file creation*

<img width="1047" height="733" alt="image" src="https://github.com/user-attachments/assets/f60d42fc-86b1-426a-90c3-9398b93de0e6" />

The lock file `.terraform.lock.hcl` was created, pinning the provider binary hash for reproducible runs.

---

### Step 6 - Patch the Instance Type Value

A stray URL artifact was detected in the `instance_type` field value within `main.tf` (likely introduced by the editing context rendering `t3.small.search` as a hyperlink string). A targeted `sed` substitution was applied to sanitize the value to the bare instance type string.

```bash
sed -i 's|t3.small.search (http://t3.small.search)||g' ~/terraform/main.tf
```

*Screenshot: sed command patching the instance_type field*

<img width="1045" height="743" alt="image" src="https://github.com/user-attachments/assets/f56a3e3c-4717-416d-82f7-88dfbbe4a951" />

> This step is a direct artifact of the tooling environment that rendered the instance type identifier as an inline URL. In a standard editor workflow, this artifact would not be present. The fix ensures `terraform validate` operates on clean HCL syntax.

---

### Step 7 - Validate the Configuration

`terraform validate` was executed to confirm HCL syntax correctness and provider schema compliance prior to any plan or apply operation.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

*Screenshot: terraform validate confirming configuration is valid*

<img width="1043" height="509" alt="image" src="https://github.com/user-attachments/assets/c1f4b5c2-6774-403b-8d61-47f0fa0734ac" />

---

### Step 8 - Apply the Infrastructure

`terraform apply -auto-approve` was executed to provision the OpenSearch domain against LocalStack. The `aws_opensearch_domain` resource has a known creation latency due to LocalStack simulating the OpenSearch domain initialization lifecycle.

```bash
terraform apply -auto-approve
```

**Execution plan summary:**

```
Terraform will perform the following actions:

  # aws_opensearch_domain.nautilus_es will be created
  + resource "aws_opensearch_domain" "nautilus_es" {
      + domain_name    = "nautilus-es"
      + engine_version = "OpenSearch_1.3"
      + cluster_config {
          + instance_type          = "t3.small.search"
          + instance_count         = 1
          + zone_awareness_enabled = false
        }
      + ebs_options {
          + ebs_enabled = true
          + volume_size = 10
          + volume_type = "gp2"
        }
      + access_policies = (jsonencode...)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Apply result:**

```
aws_opensearch_domain.nautilus_es: Creating...
aws_opensearch_domain.nautilus_es: Still creating... [10s elapsed]
...
aws_opensearch_domain.nautilus_es: Creation complete after 10m0s [id=arn:aws:es:us-east-1:000000000000:domain/nautilus-es]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

*Screenshots: Full terraform apply output showing creation progress and completion after 10 minutes*


<img width="1051" height="754" alt="image" src="https://github.com/user-attachments/assets/65d2dda1-c781-4e25-a368-9c561f158f62" />
<img width="1041" height="774" alt="image" src="https://github.com/user-attachments/assets/b63ccd0e-0854-41eb-91c1-7b9e227e1e90" />
<img width="1045" height="766" alt="image" src="https://github.com/user-attachments/assets/d056bd4c-8563-48a3-ad8d-51b18a8217c7" />
<img width="1047" height="768" alt="image" src="https://github.com/user-attachments/assets/ddc80664-9dfb-47c5-9010-566b5f0cccd3" />

The domain ARN `arn:aws:es:us-east-1:000000000000:domain/nautilus-es` confirms successful registration within the LocalStack account namespace.

---

### Step 9 - Confirm Infrastructure Convergence

`terraform plan` was re-executed immediately after apply to confirm zero drift between the Terraform state and the live infrastructure. A clean `No changes` result validates that the applied configuration is authoritative.

```bash
terraform plan
```

**Output:**

```
aws_opensearch_domain.nautilus_es: Refreshing state... [id=arn:aws:es:us-east-1:000000000000:domain/nautilus-es]

No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration
and found no differences, so no changes are needed.
```

*Screenshot: terraform plan output confirming No changes after apply*

<img width="1046" height="761" alt="image" src="https://github.com/user-attachments/assets/29cb79fb-9698-40b3-8058-fc30d81e61de" />

---

### Step 10 - Verify the Domain via AWS CLI

The domain was independently verified using the AWS CLI targeting the LocalStack endpoint, confirming that the domain attributes match the Terraform specification.

```bash
aws --endpoint-url=http://aws:4566 opensearch describe-domain --domain-name nautilus-es
```

**Output (abbreviated for clarity):**

```json
{
    "DomainStatus": {
        "DomainId": "000000000000/nautilus-es",
        "DomainName": "nautilus-es",
        "ARN": "arn:aws:es:us-east-1:000000000000:domain/nautilus-es",
        "Created": true,
        "Deleted": false,
        "Endpoint": "nautilus-es.us-east-1.opensearch.localhost.localstack.cloud:4566",
        "Processing": false,
        "EngineVersion": "OpenSearch_1.3",
        "ClusterConfig": {
            "InstanceType": "t3.small.search",
            "InstanceCount": 1,
            "DedicatedMasterEnabled": false,
            "ZoneAwarenessEnabled": false
        },
        "EBSOptions": {
            "EBSEnabled": true,
            "VolumeType": "gp2",
            "VolumeSize": 10
        },
        "AccessPolicies": "{\"Statement\":[{\"Action\":\"es:*\",\"Effect\":\"Allow\",\"Principal\":\"*\",\"Resource\":\"*\"}],\"Version\":\"2012-10-17\"}",
        "DomainProcessingStatus": "Active"
    }
}
```

*Screenshots: AWS CLI describe-domain output confirming domain is Active with correct configuration*

<img width="1050" height="770" alt="image" src="https://github.com/user-attachments/assets/5aef4c9d-c679-43a0-9765-bb9ed54e52ce" />
<img width="1045" height="776" alt="image" src="https://github.com/user-attachments/assets/a95401a9-4f04-4b08-8627-01b492a8c6f6" />
<img width="1052" height="497" alt="image" src="https://github.com/user-attachments/assets/18eef677-f3f1-4775-8284-36199675b95a" />

Key attributes verified against the specification:

* `DomainName`: `nautilus-es` - matches requirement
* `EngineVersion`: `OpenSearch_1.3` - matches requirement
* `Created`: `true`, `Deleted`: `false`, `Processing`: `false` - domain is fully active
* `DomainProcessingStatus`: `Active` - no pending operations
* `EBSOptions`: `gp2 / 10 GiB` - matches resource block
* `ClusterConfig`: `t3.small.search / 1 node / ZoneAwareness: false` - matches resource block

---

### Step 11 - Confirm Terraform State

The Terraform state file was queried to confirm the resource is correctly tracked under the expected resource address.

```bash
terraform state list
```

**Output:**

```
aws_opensearch_domain.nautilus_es
```

*Screenshot: terraform state list confirming resource is tracked in state*

<img width="1044" height="376" alt="image" src="https://github.com/user-attachments/assets/0381ed09-6077-4df9-963b-b2003375ef2c" />

The resource address `aws_opensearch_domain.nautilus_es` matches the logical name defined in `main.tf`, confirming state integrity.

---

## Resource Specification

| Attribute | Value |
|---|---|
| Resource Type | `aws_opensearch_domain` |
| Terraform Resource Name | `nautilus_es` |
| Domain Name | `nautilus-es` |
| Engine Version | `OpenSearch_1.3` |
| Instance Type | `t3.small.search` |
| Instance Count | `1` |
| Zone Awareness | Disabled |
| Dedicated Master | Disabled |
| EBS Enabled | `true` |
| Volume Type | `gp2` |
| Volume Size | `10 GiB` |
| Access Policy | `es:* / Allow / Principal: *` |
| ARN | `arn:aws:es:us-east-1:000000000000:domain/nautilus-es` |
| Endpoint | `nautilus-es.us-east-1.opensearch.localhost.localstack.cloud:4566` |

---

## Validation Results

| Validation Step | Command | Result |
|---|---|---|
| HCL syntax and schema | `terraform validate` | `Success! The configuration is valid.` |
| Infrastructure apply | `terraform apply -auto-approve` | `Apply complete! Resources: 1 added, 0 changed, 0 destroyed.` |
| Drift detection | `terraform plan` | `No changes. Your infrastructure matches the configuration.` |
| AWS-level domain status | `aws opensearch describe-domain` | `"DomainProcessingStatus": "Active"` |
| State integrity | `terraform state list` | `aws_opensearch_domain.nautilus_es` |

---

## Best Practices Applied

**Single-file resource authoring.** The task constraint required all infrastructure to be defined in `main.tf` only, with no additional `.tf` files. This enforces a minimal, auditable configuration surface for the provisioning task.

**Heredoc for file creation.** Using `cat > file << 'EOF'` with a quoted delimiter prevents variable expansion and shell interpolation, ensuring the HCL content is written verbatim. This is the safest method for programmatic file creation in shell environments.

**Pre-apply validation gate.** `terraform validate` was executed before `terraform apply` to catch schema and syntax errors without incurring any API calls. This is a standard discipline that eliminates a class of apply-time failures entirely.

**Post-apply drift check.** Re-running `terraform plan` after a successful apply is a professional verification habit that confirms Terraform's view of state matches the provider's reported resource attributes. A non-zero plan after a fresh apply is a red flag for provider bugs or LocalStack quirks.

**Independent CLI verification.** The domain was verified using the AWS CLI independently of Terraform. This cross-tool validation confirms that the resource exists at the control plane level and is not solely an artifact of Terraform state, which can become stale or out-of-sync.

**`jsonencode` for access policies.** Using Terraform's native `jsonencode()` function for the `access_policies` argument avoids JSON string escaping errors and produces readable, diff-friendly HCL rather than an opaque inline JSON string.

**Targeting LocalStack endpoints explicitly.** The `--endpoint-url=http://aws:4566` flag on all AWS CLI verification commands ensures commands route to LocalStack and do not fall back to real AWS endpoints, maintaining environment isolation.

---

## Errors Encountered and Resolutions

### Instance Type Field Contamination

**Symptom:** After the initial `cat > main.tf` write, the `instance_type` field contained a URL artifact appended to the value, rendering as `t3.small.search (http://t3.small.search)` instead of the bare `t3.small.search` string.

**Root cause:** The editing environment interpreted the instance type identifier as a hyperlinked token and injected a parenthesized URL when the content was written to file.

**Resolution:** A targeted `sed -i` substitution was applied to strip the parenthesized URL fragment:

```bash
sed -i 's|t3.small.search (http://t3.small.search)||g' ~/terraform/main.tf
```

`terraform validate` was then re-executed to confirm the field was clean before proceeding to apply.

**Lesson:** Always verify file contents with `cat` after programmatic writes and before executing `terraform validate`. The validate step acts as a final gate that would have caught this malformed value if the file inspection had been skipped.

---

## Lessons Learned

**OpenSearch domain creation latency is significant even in LocalStack.** The apply took approximately 10 minutes for the domain to reach a `Created: true` state. In real AWS environments, this latency can extend to 15 to 30 minutes. Any CI/CD pipeline that provisions OpenSearch domains must account for this and use appropriate `terraform apply` timeouts or asynchronous polling patterns.

**`skip_credentials_validation` is a LocalStack-specific provider flag, not a security bypass.** Newer engineers reviewing the `provider.tf` may flag this as a security concern. It is architecturally safe in LocalStack environments because no real credential chain is involved. In production AWS configurations, this flag must never appear.

**`aws_opensearch_domain` uses the `es` endpoint alias in LocalStack, not `opensearch`.** The provider configuration maps the `es` endpoint key to `http://aws:4566`. Despite the resource type being `aws_opensearch_domain`, the underlying API surface in older provider versions and LocalStack still routes through the Elasticsearch service endpoint namespace. This is a non-obvious mapping that is worth documenting for teams setting up new LocalStack configurations.

**Terraform state verification is not optional.** The `terraform state list` command after apply confirms that Terraform correctly registered the resource in its state backend. An absent or misnamed state entry can cause destructive behavior on subsequent runs, as Terraform would treat a managed resource as untracked and attempt to recreate it.

**`jsonencode()` produces deterministic JSON output.** Because Terraform's `jsonencode()` function always produces alphabetically ordered keys and consistent whitespace, subsequent plan runs will not produce false diffs due to JSON key ordering differences. This is preferable to embedding raw JSON strings in HCL.

---

## Technologies Used

| Technology | Version | Role |
|---|---|---|
| Terraform | v1.x | Infrastructure provisioning and state management |
| hashicorp/aws Provider | 5.91.0 | AWS resource type definitions and API translation |
| LocalStack | Latest | Local AWS emulation at `http://aws:4566` |
| Amazon OpenSearch Service | OpenSearch_1.3 | Search and analytics domain engine |
| AWS CLI | Latest | Independent domain verification and attribute inspection |
| Bash | System | File authoring, sed patching, and command execution |

---
