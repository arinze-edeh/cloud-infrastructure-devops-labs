# AWS Secrets Manager Secret Provisioning with Terraform

Provisioning a secure, versioned secret in AWS Secrets Manager using Terraform IaC on a LocalStack-backed environment, with full lifecycle validation through the AWS CLI.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Technologies Used](#technologies-used)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation](#implementation)
  - [Phase 1: Environment Verification](#phase-1-environment-verification)
  - [Phase 2: Inspect the Existing Provider Configuration](#phase-2-inspect-the-existing-provider-configuration)
  - [Phase 3: Author the Terraform Configuration](#phase-3-author-the-terraform-configuration)
  - [Phase 4: Initialize the Terraform Working Directory](#phase-4-initialize-the-terraform-working-directory)
  - [Phase 5: Validate the Configuration](#phase-5-validate-the-configuration)
  - [Phase 6: Review the Execution Plan](#phase-6-review-the-execution-plan)
  - [Phase 7: Apply and Provision Resources](#phase-7-apply-and-provision-resources)
  - [Phase 8: Validate the Deployed Secret](#phase-8-validate-the-deployed-secret)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)

---

## Overview

This project demonstrates the end-to-end provisioning of an AWS Secrets Manager secret using Terraform. The secret stores structured JSON credentials that can be consumed at runtime by application workloads. The environment uses LocalStack as an AWS API emulator, enabling full infrastructure-as-code workflows without incurring live AWS costs or requiring a production account.

---

## Problem Statement

The Nautilus DevOps team required a repeatable, auditable mechanism to create and version sensitive credentials in AWS Secrets Manager. Manual secret creation through the AWS Console introduces human error, is not reproducible across environments, and cannot be peer-reviewed through standard pull request workflows. A Terraform-based approach was needed to enforce consistency, traceability, and team handoff.

**Requirements:**

* Secret name: `xfusion-secret`
* Secret value: a JSON key-value pair containing `username: admin` and `password: Namin123`
* Provisioning tool: Terraform only, using a single `main.tf` file
* Working directory: `/home/bob/terraform`

---

## Solution Architecture

```
Developer Workstation
        |
        v
  Terraform CLI (v1.11.0)
        |
        v
  hashicorp/aws Provider (v5.91.0)
        |
        v
  LocalStack (http://aws:4566)
        |
        v
  AWS Secrets Manager (emulated)
        |
   xfusion-secret
        |
   SecretVersion (AWSCURRENT)
        |-- username: admin
        |-- password: Namin123
```

Two Terraform resources are provisioned in dependency order:

1. `aws_secretsmanager_secret` -- creates the secret container with the specified name
2. `aws_secretsmanager_secret_version` -- attaches a versioned JSON string payload to the secret, referencing the container by resource reference

---

## Technologies Used

| Technology | Version | Role |
|---|---|---|
| Terraform | v1.11.0 | Infrastructure provisioning and state management |
| hashicorp/aws Provider | v5.91.0 | Terraform AWS resource driver |
| AWS Secrets Manager | N/A | Secure secret storage service |
| LocalStack | Latest | Local AWS API emulator (endpoint: `http://aws:4566`) |
| AWS CLI | Latest | Post-apply secret retrieval and validation |
| Bash | System | Command execution and heredoc authoring |

---

## Prerequisites

* Terraform v1.11.0 or later installed
* AWS CLI configured and targeting the LocalStack endpoint
* LocalStack running and accessible at `http://aws:4566`
* Write access to `/home/bob/terraform`
* An existing `provider.tf` configuring the `hashicorp/aws` provider with LocalStack endpoint overrides

---

## Repository Structure

```
/home/bob/terraform/
    provider.tf       # AWS provider configuration with LocalStack endpoint mapping
    main.tf           # Secret and secret version resource definitions (authored in this project)
    README.MD         # Existing project readme
    .terraform/       # Provider plugins (generated after init)
    .terraform.lock.hcl  # Provider dependency lock file (generated after init)
```

---

## Implementation

### Phase 1: Environment Verification

Before authoring any configuration, the Terraform version and AWS identity were confirmed to establish a known-good baseline.

**Verify Terraform version:**

```bash
terraform version
```

**Output:**

```
Terraform v1.11.0
on linux_amd64

Your version of Terraform is out of date! The latest version
is 1.14.9. You can update by downloading from https://www.terraform.io/downloads.html
```

> Screenshot: Terraform version output showing v1.11.0 on linux/amd64

<img width="1028" height="600" alt="image" src="https://github.com/user-attachments/assets/edd48fce-c149-4914-be1c-9eba6a997c27" />

**Verify AWS CLI identity and LocalStack connectivity:**

```bash
aws sts get-caller-identity
```

**Output:**

```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

> Screenshot: AWS STS caller identity output confirming LocalStack connectivity

<img width="1031" height="691" alt="image" src="https://github.com/user-attachments/assets/a1762274-8d64-4d90-ad4f-53c5d02cfefc" />

The `Account: 000000000000` value confirms the CLI is routed through LocalStack rather than a live AWS account.

**List existing working directory contents:**

```bash
ls -la
```

**Output:**

```
total 20
drwxr-xr-x 1 bob bob 4096 Jun 19  2025 .
drwxr-x--- 1 bob bob 4096 Apr 26 09:06 ..
-rw-rw-r-- 1 bob bob  435 Jun 19  2025 README.MD
-rw-rw-r-- 1 bob bob 1116 May 13  2025 provider.tf
```

> Screenshot: Directory listing confirming only provider.tf and README.MD exist before implementation

<img width="1021" height="667" alt="image" src="https://github.com/user-attachments/assets/75f95602-dee8-4023-8e75-47495033b852" />

---

### Phase 2: Inspect the Existing Provider Configuration

The pre-existing `provider.tf` was reviewed to understand the LocalStack endpoint mapping and confirm the AWS region before authoring `main.tf`.

```bash
cat provider.tf
```

**Output:**

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

> Screenshot: cat output of provider.tf showing LocalStack endpoint overrides including secretsmanager at http://aws:4566

<img width="1071" height="797" alt="image" src="https://github.com/user-attachments/assets/16c8099f-a216-4e99-a689-2e7025b0012e" />

Key observations:

* `skip_credentials_validation = true` and `skip_requesting_account_id = true` are required for LocalStack compatibility
* The `secretsmanager` endpoint is already mapped to `http://aws:4566`, confirming that `aws_secretsmanager_secret` resources will route correctly
* `s3_use_path_style = true` is a LocalStack-specific requirement for S3 path addressing

---

### Phase 3: Author the Terraform Configuration

The `main.tf` file was created in the Terraform working directory using a bash heredoc, keeping all secret infrastructure scoped to a single file as required.

```bash
cat > /home/bob/terraform/main.tf << 'EOF'
resource "aws_secretsmanager_secret" "xfusion_secret" {
  name = "xfusion-secret"
}

resource "aws_secretsmanager_secret_version" "xfusion_secret_version" {
  secret_id = aws_secretsmanager_secret.xfusion_secret.id

  secret_string = jsonencode({
    username = "admin"
    password = "Namin123"
  })
}
EOF
```

**Verify the written file:**

```bash
cat /home/bob/terraform/main.tf
```

**Output:**

```hcl
resource "aws_secretsmanager_secret" "xfusion_secret" {
  name = "xfusion-secret"
}

resource "aws_secretsmanager_secret_version" "xfusion_secret_version" {
  secret_id = aws_secretsmanager_secret.xfusion_secret.id

  secret_string = jsonencode({
    username = "admin"
    password = "Namin123"
  })
}
```

> Screenshot: cat output of main.tf confirming the two resource blocks were written correctly

<img width="1046" height="740" alt="image" src="https://github.com/user-attachments/assets/ed5bd41d-0638-4d91-8a27-9f1c1d82d3e1" />

**Design decisions:**

* `jsonencode()` is used instead of a raw string literal to ensure the secret value is always valid, machine-parseable JSON regardless of special characters in the credential values
* `secret_id = aws_secretsmanager_secret.xfusion_secret.id` establishes an implicit resource dependency, guaranteeing Terraform provisions the secret container before the version
* No `description`, `kms_key_id`, or `recovery_window_in_days` overrides were set, relying on service defaults appropriate for a LocalStack environment

---

### Phase 4: Initialize the Terraform Working Directory

The working directory was initialized, which downloads the `hashicorp/aws` provider plugin and creates the lock file.

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

> Screenshot: terraform init output confirming hashicorp/aws v5.91.0 was installed and the backend was initialized

<img width="1044" height="656" alt="image" src="https://github.com/user-attachments/assets/853a4d21-70e0-4ab5-be8a-df02f18ae12e" />

The `.terraform.lock.hcl` file pins the provider to `v5.91.0`. This file must be committed to version control to guarantee reproducible provider resolution across team members and CI/CD pipelines.

---

### Phase 5: Validate the Configuration

Terraform's built-in configuration validator was run to detect any syntax or schema errors before planning.

```bash
terraform validate
```

**Output:**

```
Success! The configuration is valid.
```

> Screenshot: terraform validate showing "Success! The configuration is valid."

<img width="1049" height="477" alt="image" src="https://github.com/user-attachments/assets/73678414-4fc2-4732-b819-2e097a8e8817" />

Validation confirms that:

* All resource blocks reference valid resource types in the installed provider
* All required arguments are present
* The `jsonencode()` function call is syntactically correct
* The `secret_id` reference resolves to a known resource attribute

---

### Phase 6: Review the Execution Plan

A dry-run plan was generated to inspect all resource actions before any infrastructure was modified.

```bash
terraform plan
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_secretsmanager_secret.xfusion_secret will be created
  + resource "aws_secretsmanager_secret" "xfusion_secret" {
      + arn                            = (known after apply)
      + force_overwrite_replica_secret = false
      + id                             = (known after apply)
      + name                           = "xfusion-secret"
      + name_prefix                    = (known after apply)
      + policy                         = (known after apply)
      + recovery_window_in_days        = 30
      + tags_all                       = (known after apply)

      + replica (known after apply)
    }

  # aws_secretsmanager_secret_version.xfusion_secret_version will be created
  + resource "aws_secretsmanager_secret_version" "xfusion_secret_version" {
      + arn                  = (known after apply)
      + has_secret_string_wo = (known after apply)
      + id                   = (known after apply)
      + secret_id            = (known after apply)
      + secret_string        = (sensitive value)
      + secret_string_wo     = (write-only attribute)
      + version_id           = (known after apply)
      + version_stages       = (known after apply)
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

> Screenshot: terraform plan output showing two resources to be created with no changes or destructions planned

Key plan observations:

* `recovery_window_in_days = 30` confirms the default deletion protection window is active
* `secret_string = (sensitive value)` confirms Terraform masks the credential payload in plan output, preventing accidental exposure in CI/CD logs
* `Plan: 2 to add, 0 to change, 0 to destroy` confirms a clean net-new provisioning with no side effects on existing state

---

### Phase 7: Apply and Provision Resources

The plan was applied with auto-approval to execute the provisioning without an interactive confirmation prompt.

```bash
terraform apply -auto-approve
```

**Output:**

```
Terraform used the selected providers to generate the following execution plan.
...
Plan: 2 to add, 0 to change, 0 to destroy.
aws_secretsmanager_secret.xfusion_secret: Creating...
aws_secretsmanager_secret.xfusion_secret: Creation complete after 0s [id=arn:aws:secretsmanager:us-east-1:000000000000:secret:xfusion-secret-halazl]
aws_secretsmanager_secret_version.xfusion_secret_version: Creating...
aws_secretsmanager_secret_version.xfusion_secret_version: Creation complete after 0s [id=arn:aws:secretsmanager:us-east-1:000000000000:secret:xfusion-secret-halazl|terraform-20260426091515669900000002]

Apply complete! Resources: 2 added, 0 changed, 0 destroyed.
```

> Screenshot: terraform apply -auto-approve output showing both resources created successfully with their ARNs

**Provisioned resource identifiers:**

| Resource | ARN / ID |
|---|---|
| `aws_secretsmanager_secret.xfusion_secret` | `arn:aws:secretsmanager:us-east-1:000000000000:secret:xfusion-secret-halazl` |
| `aws_secretsmanager_secret_version.xfusion_secret_version` | `...xfusion-secret-halazl|terraform-20260426091515669900000002` |

The `-halazl` suffix in the ARN is a random identifier appended by Secrets Manager to ensure global uniqueness. This is expected behavior for the service.

---

### Phase 8: Validate the Deployed Secret

Two AWS CLI commands were used to independently verify the provisioned secret, confirming both metadata and payload integrity.

**Retrieve secret metadata:**

```bash
aws secretsmanager describe-secret --secret-id xfusion-secret
```

**Output:**

```json
{
    "ARN": "arn:aws:secretsmanager:us-east-1:000000000000:secret:xfusion-secret-halazl",
    "Name": "xfusion-secret",
    "LastChangedDate": 1777194915.672,
    "LastAccessedDate": 1777161600.0,
    "VersionIdsToStages": {
        "terraform-20260426091515669900000002": [
            "AWSCURRENT"
        ]
    },
    "CreatedDate": 1777194915.662362
}
```

> Screenshot: aws secretsmanager describe-secret output showing the secret name, ARN, version ID, and AWSCURRENT stage label

**Retrieve and decode the secret value:**

```bash
aws secretsmanager get-secret-value --secret-id xfusion-secret --query SecretString --output text
```

**Output:**

```json
{"password":"Namin123","username":"admin"}
```

> Screenshot: aws secretsmanager get-secret-value output confirming the JSON payload contains the correct username and password values

Both validation commands confirm:

* The secret name `xfusion-secret` resolves correctly
* The active version is labeled `AWSCURRENT`, confirming it is the live, routable version for consuming applications
* The JSON payload contains the expected `username` and `password` key-value pairs

---

## Best Practices Applied

**Implicit resource dependencies via attribute references**
Using `secret_id = aws_secretsmanager_secret.xfusion_secret.id` rather than a `depends_on` block ensures Terraform resolves the creation order through its dependency graph, keeping the code idiomatic and readable.

**jsonencode() for structured secret payloads**
Encoding credentials as JSON using the `jsonencode()` built-in function ensures the secret string is always valid JSON, enabling consuming applications to use standard JSON parsers without defensive error handling.

**Sensitive value masking in plan output**
Terraform automatically masks `secret_string` in plan and apply output as `(sensitive value)`. This prevents credential exposure in CI/CD pipeline logs and terminal sessions, maintaining security hygiene without additional configuration.

**Provider lock file committed to version control**
The `.terraform.lock.hcl` file pins the `hashicorp/aws` provider to `v5.91.0`. Committing this file ensures all team members and CI/CD runners use an identical provider binary, eliminating provider drift across environments.

**Validation before apply**
Running `terraform validate` before `terraform plan` catches schema and syntax errors early, before the provider attempts any API calls, reducing wasted time on avoidable apply failures.

**Heredoc for file authoring**
Using a bash heredoc (`<< 'EOF'`) to write `main.tf` ensures exact content fidelity, avoids shell interpolation of special characters, and provides an auditable, copy-paste-reproducible authoring step.

---

## Lessons Learned

**LocalStack endpoint mapping must cover every service in use**
The `provider.tf` must explicitly map each AWS service endpoint to the LocalStack host. If `secretsmanager` were absent from the `endpoints` block, all Terraform API calls to Secrets Manager would fail with connection errors rather than a clear configuration message. Always audit the endpoint map against every service referenced in `main.tf` before applying.

**The `-halazl` ARN suffix is expected, not an error**
AWS Secrets Manager appends a random six-character suffix to the ARN of every secret to enforce global uniqueness. This suffix appears in the ARN but does not affect lookups by secret name. Applications and scripts should reference secrets by name, not by full ARN, to remain portable.

**`secret_string` masking applies only to Terraform output, not to state files**
While Terraform masks `secret_string` in plan and apply terminal output, the raw credential value is stored in plaintext within `terraform.tfstate`. In production environments, the state file must be stored in an encrypted remote backend such as S3 with SSE-KMS, with access restricted to authorized principals. Using a local state file for workloads with real credentials is not acceptable.

**`recovery_window_in_days = 30` prevents immediate secret deletion**
The default recovery window means that a `terraform destroy` will schedule the secret for deletion rather than removing it immediately. In LocalStack environments this is generally inconsequential, but in production it means re-creating a secret with the same name within the 30-day window will fail unless `recovery_window_in_days = 0` is explicitly set or `force_overwrite_replica_secret = true` is used.

**`terraform apply -auto-approve` is appropriate for automated pipelines, not ad-hoc production changes**
The `-auto-approve` flag bypasses the interactive confirmation prompt. In a CI/CD context this is required. For ad-hoc changes in production environments, omitting the flag and reviewing the plan output before confirming is the safer pattern.

---

## Outcome

A fully versioned AWS Secrets Manager secret named `xfusion-secret` was provisioned through Terraform IaC in a LocalStack environment. The secret contains a JSON payload with `username: admin` and `password: Namin123`, staged as `AWSCURRENT` and immediately available for consumption by authorized workloads. The end-to-end workflow from environment verification through post-apply validation was completed without errors, producing a reproducible, peer-reviewable infrastructure definition in `main.tf`.

| Metric | Result |
|---|---|
| Resources provisioned | 2 |
| Resources changed | 0 |
| Resources destroyed | 0 |
| Secret stage | AWSCURRENT |
| Payload format | JSON (via jsonencode) |
| Validation method | AWS CLI describe-secret + get-secret-value |







<img width="1068" height="653" alt="image" src="https://github.com/user-attachments/assets/fa161fda-b0b9-40a9-b863-666fdf005951" />
<img width="1077" height="817" alt="image" src="https://github.com/user-attachments/assets/fe8aa0e6-2208-4b42-bd54-f31216fe404c" />
<img width="1047" height="462" alt="image" src="https://github.com/user-attachments/assets/5692742a-74f0-4952-8174-14460eef0245" />
<img width="1047" height="529" alt="image" src="https://github.com/user-attachments/assets/d98384bb-752a-4c66-b8b8-a16688c81e30" />


