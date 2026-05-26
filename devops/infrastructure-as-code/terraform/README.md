# Terraform Infrastructure as Code Portfolio

> AWS resource provisioning and lifecycle management using Terraform, targeting both LocalStack-emulated and production-style AWS environments. All tasks reflect real-world DevOps workflows: phased cloud migration, IAM governance, network provisioning, storage automation, monitoring, and controlled decommissioning.

---

## Overview

This directory contains 40+ discrete Terraform projects executed as part of the **Nautilus DevOps** cloud migration initiative. Each project represents a production-relevant infrastructure unit, provisioned using the full Terraform lifecycle: `init > validate > plan > apply > verify`.

The work spans core AWS service domains and consistently applies IaC disciplines including provider version pinning, state-driven operations, implicit dependency resolution, dual-layer verification (Terraform state + AWS CLI), and configuration preservation post-destroy.

---

## Directory Structure

```
terraform/
├── aws-aws-ec2-instance-retirement/
├── aws-cloudformation-s3-versioning/
├── aws-cloudwatch-log-provisioning/
├── aws-dynamodb-table-provisioning/
├── aws-ebs-snapshot-automation/
├── aws-ebs-volume-provisioning/
├── aws-ec2-elastic-ip-association/
├── aws-ec2-instance-resizing/
├── aws-ec2-provisioning/
├── aws-ec2-sg-provision/
├── aws-elastic-ip-allocation/
├── aws-elastic-ip/
├── aws-iam-group-decommission/
├── aws-iam-group-provisioning/
├── aws-iam-policy-forge/
├── aws-iam-user-managed-policy-attachment/
├── aws-key-pair-provisioning/
├── aws-kinesis-stream-provisioning/
├── aws-opensearch-domain-provisioning/
├── aws-s3-backup-and-bucket-cleanup/
├── aws-s3-bucket-versioning/
├── aws-s3-file-migration/
├── aws-s3-private-bucket-provisioning/
├── aws-s3-public-bucket-provisioning/
├── aws-secrets-manager-provisioning/
├── aws-security-group-provisioning/
├── aws-sns-topic-provisioning/
├── aws-ssm-parameter-store/
├── aws-vpc-datacenter-provisioning/
├── aws-vpc-init/
├── aws-vpc-ipv6-provisioning/
├── aws-vpc-lifecycle-decommission/
├── aws-vpc-provisioning/
├── cloudwatch-ec2-cpu-alarm/
├── ec2-ami-provisioning/
├── iam-role-decommission/
├── iam-user-provisioning/
└── README.md
```

---

## Project Summaries

### Compute

---

#### [EC2 Instance Provisioning](./aws-ec2-provisioning/)

**Quick summary:** Provision a `t2.micro` EC2 instance against LocalStack using a `data` source for dynamic security group resolution. Resolves a duplicate provider conflict introduced by a pre-existing `provider.tf`.

**Purpose:** Bootstrap compute infrastructure for the Nautilus migration while demonstrating the correct handling of shared Terraform workspaces.

**Approach:** Used a `data "aws_security_group"` lookup instead of hardcoding the security group ID. Identified and resolved a `Duplicate provider configuration` error by stripping conflicting blocks from `main.tf` after auditing the pre-existing `provider.tf`.

**Outcome:** Instance `datacenter-ec2` provisioned with AMI `ami-0c101f26f147fa7fd`, key pair `datacenter-kp`, and default VPC security group. Full `init > validate > plan > apply` lifecycle completed.

---

#### [EC2 Instance Type Modification](./aws-ec2-instance-resizing/)

**Quick summary:** Resize a running EC2 instance from `t2.micro` to `t2.nano` using Terraform's in-place update workflow without resource replacement.

**Purpose:** Right-size underutilised compute during active migration to reduce cost.

**Approach:** Applied a targeted `sed -i` substitution to `main.tf`, then verified the plan showed `~ update in-place` (not `-/+` replace) before applying. Confirmed with both `terraform show` and `aws ec2 describe-instances`.

**Outcome:** Instance `datacenter-ec2` resized in 20 seconds, instance ID preserved, confirmed `t2.nano` running state via AWS CLI.

---

#### [EC2 Instance Retirement (Targeted Destroy)](./aws-aws-ec2-instance-retirement/)

**Quick summary:** Destroy a single EC2 instance via `terraform destroy -target` while preserving the provisioning configuration for future re-use.

**Purpose:** Decommission a migration test resource cleanly through Terraform to maintain state consistency.

**Approach:** Ran `terraform plan -destroy -target=aws_instance.ec2` for a pre-destroy audit, confirmed the instance ID against AWS CLI, then executed the targeted destroy. Validated with `aws ec2 describe-instances --filters` returning `terminated` state.

**Outcome:** Instance `nautilus-ec2` terminated, state cleared, `main.tf` preserved intact. Post-destroy `terraform plan` confirms re-provisioning readiness.

---

#### [EC2 AMI Creation from Running Instance](./ec2-ami-provisioning/)

**Quick summary:** Create a golden AMI from a running EC2 instance by appending an `aws_ami_from_instance` resource to an existing `main.tf`.

**Purpose:** Capture instance state as a reusable image to support environment replication across migration phases.

**Approach:** Referenced the source instance dynamically via `aws_instance.ec2.id` to establish an implicit dependency. Verified the AMI state is `available` using a filtered `aws ec2 describe-images` call.

**Outcome:** AMI `nautilus-ec2-ami` created in 5 seconds, confirmed `available` via AWS CLI.

---

#### [EC2 Key Pair Provisioning](./aws-key-pair-provisioning/)

**Quick summary:** Generate an RSA 4096-bit key pair using `hashicorp/tls`, register the public key with EC2 via `aws_key_pair`, and persist the private key locally with `local_file` at `0400` permissions.

**Purpose:** Establish SSH access credentials for EC2 instances as a prerequisite to compute provisioning.

**Approach:** Used the `tls_private_key` resource so private key material never transits the AWS API. Three-provider configuration (`aws`, `tls`, `local`) resolved implicitly via attribute references.

**Outcome:** Key pair `nautilus-kp` registered, private key `nautilus-kp.pem` written with owner-only read permissions, confirmed via `terraform state show`.

---

### Networking

---

#### [VPC Provisioning](./aws-vpc-provisioning/)

**Quick summary:** Provision a custom VPC named `devops-vpc` with CIDR `10.0.0.0/16` in `us-east-1` as the foundational networking layer for the migration.

**Purpose:** Establish the network isolation boundary before onboarding services.

**Approach:** Minimal `main.tf` with a single `aws_vpc` resource. Post-apply `terraform show` confirmed computed attributes: default ACL, route table, security group, and DHCP options auto-created by AWS.

**Outcome:** VPC provisioned. DNS support enabled by default; `enable_dns_hostnames` noted as requiring explicit opt-in for instance public DNS resolution in later phases.

---

#### [VPC with IPv6 Support](./aws-vpc-ipv6-provisioning/)

**Quick summary:** Extend VPC provisioning with `assign_generated_ipv6_cidr_block = true` to support dual-stack networking.

**Purpose:** Prepare the VPC for modern IPv6-capable workloads.

**Outcome:** Amazon-assigned `/56` IPv6 CIDR allocated, `ipv6_association_id` populated and confirmed in state.

---

#### [VPC Variable-Driven Provisioning](./aws-vpc-init/)

**Quick summary:** Provision a VPC with the `Name` tag sourced from input variable `KKE_vpc`, separating configuration values from resource logic.

**Purpose:** Enforce modularity and enable multi-environment reuse without modifying resource code.

**Outcome:** VPC `xfusion-vpc` provisioned with CIDR `10.0.0.0/16`. Variable override via `-var` or `.tfvars` confirmed operable.

---

#### [VPC Decommission](./aws-vpc-lifecycle-decommission/)

**Quick summary:** Destroy a managed VPC using `terraform destroy`, verify the state file is empty, and confirm the provisioning code is preserved for future re-use.

**Outcome:** `terraform.tfstate` reduced to an empty `resources` array. `main.tf` confirmed intact. `terraform.tfstate.backup` auto-generated as a pre-destroy snapshot.

---

#### [Security Group Provisioning](./aws-ec2-sg-provision/)

**Quick summary:** Provision `nautilus-sg` with inbound rules for TCP 80 (HTTP) and TCP 22 (SSH) from `0.0.0.0/0` using the `aws_security_group` resource.

**Outcome:** Security group confirmed in state with both ingress rules and default-VPC attachment.

---

#### [Security Group with Variable Abstraction](./aws-security-group-provisioning/)

**Quick summary:** Provision `datacenter-sg` with the group name stored in input variable `KKE_sg`.

**Outcome:** `aws_security_group.datacenter_sg` created and confirmed in state.

---

#### [Elastic IP Allocation](./aws-elastic-ip/)

**Quick summary:** Allocate a VPC-domain Elastic IP using `aws_eip` with the `Name` tag sourced from input variable `KKE_eip`. Removes the deprecated `vpc = true` argument in favor of implicit v5 provider behavior.

**Outcome:** EIP allocated, `domain = "vpc"` confirmed in state.

---

#### [Elastic IP Allocation (Direct)](./aws-elastic-ip-allocation/)

**Quick summary:** Provision EIP `devops-eip` directly in `main.tf` without variable abstraction. Resolves a duplicate `provider.tf` block conflict and removes deprecated `vpc = true`.

**Outcome:** EIP allocated, public DNS confirmed.

---

#### [Elastic IP Association to EC2](./aws-ec2-elastic-ip-association/)

**Quick summary:** Bind a pre-provisioned EIP to an existing EC2 instance by appending `aws_eip_association` to `main.tf`, using attribute references rather than hardcoded IDs.

**Approach:** Pre-verified EIP was unassociated via `aws ec2 describe-addresses` before modifying configuration. Post-apply confirmed `AssociationId`, `InstanceId`, and `PublicIP` via a table-formatted CLI query.

**Outcome:** EIP bound to EC2 instance. Association confirmed via AWS CLI and `terraform state list`.

---

### Storage

---

#### [EBS Volume Provisioning](./aws-ebs-volume-provisioning/)

**Quick summary:** Provision a 2 GiB `gp3` EBS volume in `us-east-1a`. Resolves a duplicate provider conflict before completing the lifecycle.

**Outcome:** Volume created. `gp3` selected over legacy `gp2` for better price-performance.

---

#### [EBS Snapshot Automation](./aws-ebs-snapshot-automation/)

**Quick summary:** Append `aws_ebs_snapshot` to an existing `main.tf` that already tracks an EBS volume, linking them via implicit reference `aws_ebs_volume.k8s_volume.id`.

**Outcome:** Snapshot created from source volume, confirmed available with correct `volume_size` via `terraform show`.

---

#### [S3 Private Bucket Provisioning](./aws-s3-private-bucket-provisioning/)

**Quick summary:** Provision an S3 bucket with all four `aws_s3_bucket_public_access_block` controls set to `true`, using the v4+ standalone resource pattern.

**Outcome:** Both resources created in dependency order. All block flags confirmed `true` via `terraform state show`.

---

#### [S3 Public Bucket Provisioning](./aws-s3-public-bucket-provisioning/)

**Quick summary:** Provision a public S3 bucket using four coordinated resources: `aws_s3_bucket`, `aws_s3_bucket_ownership_controls` (`BucketOwnerPreferred`), `aws_s3_bucket_public_access_block` (all flags `false`), and `aws_s3_bucket_acl` (`public-read`) with explicit `depends_on`.

**Outcome:** `AllUsers` grantee with `READ` permission confirmed via `aws s3api get-bucket-acl`.

---

#### [S3 Bucket Versioning](./aws-s3-bucket-versioning/)

**Quick summary:** Enable versioning on an existing S3 bucket by appending `aws_s3_bucket_versioning` to `main.tf`, referencing the bucket via attribute reference.

**Outcome:** `Status: Enabled` confirmed via `aws s3api get-bucket-versioning`.

---

#### [S3 Object Upload (File Migration)](./aws-s3-file-migration/)

**Quick summary:** Upload a local file to an S3 bucket by appending `aws_s3_object` with `etag = filemd5(...)` for idempotent drift detection.

**Outcome:** Object confirmed in bucket via `aws s3 ls`.

---

#### [S3 Backup and Bucket Cleanup](./aws-s3-backup-and-bucket-cleanup/)

**Quick summary:** Copy all objects from an S3 bucket to a local backup directory then delete the bucket, using sequential `local-exec` provisioners on a `terraform_data` resource.

**Approach:** Two ordered `local-exec` provisioners guarantee copy-before-delete. `--force` on `aws s3 rb` provides a fallback for residual objects. `--endpoint-url` explicitly set in every shell command since `local-exec` runs as an independent process outside the provider endpoint context.

**Outcome:** Backup file confirmed in `/opt/s3-backup/`. Bucket deletion confirmed with `NoSuchBucket` from `aws s3 ls`.

---

#### [CloudFormation Stack: S3 with Versioning](./aws-cloudformation-s3-versioning/)

**Quick summary:** Provision a CloudFormation stack via `aws_cloudformation_stack`, with an inline `jsonencode()` template that creates an S3 bucket with versioning enabled.

**Purpose:** Demonstrate the Terraform-manages-CloudFormation pattern common in environments migrating from native CF to unified IaC tooling.

**Outcome:** Stack status `CREATE_COMPLETE`, bucket versioning `Enabled`, both confirmed via AWS CLI.

---

### Identity and Access Management

---

#### [IAM User Provisioning](./iam-user-provisioning/)

**Quick summary:** Provision an IAM user via `aws_iam_user`. Confirms the full lifecycle from workspace audit through AWS CLI `list-users` verification.

**Outcome:** User confirmed in LocalStack with correct ARN and path.

---

#### [IAM Group Provisioning](./aws-iam-group-provisioning/)

**Quick summary:** Provision an IAM group using `aws_iam_group`. Reviewed `provider.tf` before authoring to avoid duplicate provider blocks.

**Outcome:** Group confirmed via `terraform state show` with ARN and unique ID populated.

---

#### [IAM Group Decommission](./aws-iam-group-decommission/)

**Quick summary:** Delete an IAM group via targeted destroy, verify state is empty, confirm re-provisioning readiness via a clean `terraform plan`.

**Outcome:** `aws iam get-group` returns `NoSuchEntity`. Configuration preserved, `terraform plan` shows `1 to add`.

---

#### [IAM Policy Provisioning](./aws-iam-policy-forge/)

**Quick summary:** Create a customer-managed IAM policy granting `ec2:Describe*`, `ec2:Get*`, and `ec2:List*` using `jsonencode()` for inline policy authoring.

**Outcome:** Policy confirmed via `aws iam list-policies --scope Local`.

---

#### [IAM Policy Attachment](./aws-iam-user-managed-policy-attachment/)

**Quick summary:** Attach a managed IAM policy to an IAM user by appending `aws_iam_user_policy_attachment` to an existing `main.tf` containing both the user and policy resources.

**Outcome:** Attachment confirmed via `aws iam list-attached-user-policies`.

---

#### [IAM Role Decommission](./iam-role-decommission/)

**Quick summary:** Delete an IAM role via `terraform destroy -target`. Pre-destroy plan reviewed, post-destroy state verified empty, CLI `get-role` returns `NoSuchEntity`.

**Outcome:** Role fully deleted, state cleared, `terraform.tfstate.backup` auto-generated as a recovery snapshot.

---

### Databases and Streaming

---

#### [DynamoDB Table Provisioning](./aws-dynamodb-table-provisioning/)

**Quick summary:** Provision a DynamoDB table with a String hash key and `PAY_PER_REQUEST` billing. Resolves a duplicate `required_providers` and `provider "aws"` conflict before succeeding.

**Outcome:** Table confirmed in state. `PAY_PER_REQUEST` eliminates capacity management overhead.

---

#### [Kinesis Stream Provisioning](./aws-kinesis-stream-provisioning/)

**Quick summary:** Provision a Kinesis stream with 1 shard and 24-hour retention. Detects and resolves a tag drift loop caused by LocalStack's inconsistent `DescribeStreamSummary` response.

**Approach:** Removed the `tags` block after confirming it was not a task requirement and was causing a persistent `~ update in-place` on every subsequent plan. Verified idempotency with `No changes` on the final plan.

**Outcome:** Stream confirmed active via ARN, idempotent state achieved.

---

#### [OpenSearch Domain Provisioning](./aws-opensearch-domain-provisioning/)

**Quick summary:** Provision an OpenSearch domain (`OpenSearch_1.3`, `t3.small.search`, 10 GiB `gp2`) with a fully open access policy using `jsonencode()`.

**Outcome:** Domain confirmed `Active` via `aws opensearch describe-domain`.

---

### Monitoring

---

#### [CloudWatch Log Group and Stream](./aws-cloudwatch-log-provisioning/)

**Quick summary:** Provision a log group and log stream using implicit dependency via `log_group_name = aws_cloudwatch_log_group.name`.

**Outcome:** Both resources confirmed in `terraform state list`, creation order resolved correctly by Terraform's dependency graph.

---

#### [CloudWatch CPU Alarm](./cloudwatch-ec2-cpu-alarm/)

**Quick summary:** Provision a CloudWatch metric alarm for EC2 CPU utilization with threshold 80%, 5-minute period, and a single evaluation period.

**Outcome:** Alarm confirmed via `aws cloudwatch describe-alarms` with `StateValue: INSUFFICIENT_DATA` (expected initial state) and all metric parameters matching specification.

---

### Secrets and Parameters

---

#### [Secrets Manager Provisioning](./aws-secrets-manager-provisioning/)

**Quick summary:** Provision a versioned secret containing structured JSON credentials using `jsonencode()`. Secret value is automatically masked as `(sensitive value)` in plan output.

**Outcome:** Secret confirmed `AWSCURRENT` stage via `aws secretsmanager get-secret-value`.

---

#### [SSM Parameter Store](./aws-ssm-parameter-store/)

**Quick summary:** Provision an SSM parameter of type `String`.

**Outcome:** Parameter confirmed via `aws ssm get-parameter` with correct `Name`, `Type`, `Value`, and `Version: 1`.

---

#### [SNS Topic Provisioning](./aws-sns-topic-provisioning/)

**Quick summary:** Provision an SNS topic. Resolves a duplicate provider conflict and an AWS CLI endpoint mismatch (`localhost:4566` vs `aws:4566`) encountered during post-apply verification.

**Outcome:** Topic ARN confirmed via `aws sns list-topics`.

---

## Technologies and Tools

| Category | Tools |
|---|---|
| IaC Engine | Terraform (v1.11.0) |
| Cloud Provider | AWS (via `hashicorp/aws` v5.91.0) |
| AWS Emulation | LocalStack (`http://aws:4566`) |
| Verification | AWS CLI (v1.x / v2.x) |
| Shell | Bash (heredoc authoring, `sed`, `aws`, `terraform`) |
| Additional Providers | `hashicorp/tls`, `hashicorp/local` |

**AWS Services Covered:** EC2, VPC, EBS, S3, IAM, DynamoDB, Kinesis, OpenSearch, CloudWatch, Secrets Manager, SSM, SNS, CloudFormation

---

## Key Skills Demonstrated

**State management**
Targeted destroy with `-target`, pre/post state verification, `terraform state list` and `terraform state show`, state backup awareness, and configuration preservation post-destroy.

**Conflict resolution**
Consistent identification and resolution of `Duplicate provider configuration` and `Duplicate required providers configuration` errors caused by pre-existing `provider.tf` files in shared workspaces.

**Dependency modeling**
Implicit dependency resolution via attribute references (`resource.type.name.attribute`) across all resource pairs, avoiding unnecessary `depends_on` except where provider ordering semantics require it (e.g., S3 ACL application sequence).

**Dual-layer verification**
Every project confirms results through both Terraform state inspection and independent AWS CLI queries, separating IaC-layer assertions from provider-layer ground truth.

**Idempotency discipline**
Post-apply `terraform plan` executed on all projects to confirm `No changes`, treating convergence as the acceptance criterion rather than apply success alone.

**Production patterns applied**
Provider version pinning with lock files, `validate > plan > apply` gating on every project, `jsonencode()` for inline policy and template authoring, tag-based CLI resource filtering, and `s3_use_path_style` for LocalStack S3 compatibility.

---

## How to Navigate

Each subdirectory is a self-contained Terraform project with its own `README.md` containing the full implementation guide, configuration reference, and verification output.

**To reproduce any project:**

```bash
cd <project-directory>
cat provider.tf          # review endpoint configuration
terraform init
terraform validate
terraform plan
terraform apply -auto-approve
```

All projects require LocalStack running at `http://aws:4566`. Start LocalStack with:

```bash
localstack start
```

**To reproduce cleanly after a prior run:**

```bash
terraform destroy -auto-approve
terraform apply -auto-approve
```

---

## Author

**Arinze Edeh** | Cloud and DevOps Engineer

[GitHub](https://github.com/arinze-edeh) | [LinkedIn](https://linkedin.com/in/arinze-edeh)
