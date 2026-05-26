# AWS Cloud Engineering

![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=flat-square&logo=amazon-aws)
![IaC](https://img.shields.io/badge/IaC-CloudFormation%20%7C%20CLI-FF9900?style=flat-square&logo=amazon-aws)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python)
![Region](https://img.shields.io/badge/Region-us--east--1-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Production--Patterned-brightgreen?style=flat-square)

---

## Overview

This repository documents a structured body of AWS infrastructure engineering work spanning compute, containers, databases, networking, serverless, storage, security, observability, and automation. Every project is grounded in a real-world operational scenario: a misconfigured VPC under incident response, a private EC2 workload that needs internet egress without NAT Gateway costs, a Lambda pipeline that must drain a priority queue in strict order regardless of message arrival timing.

All implementations are CLI-first, production-patterned, and documented to the standard expected in automated pipelines and audit-sensitive environments. No console-only steps appear in the critical path. Failures encountered during implementation are documented with exact error output, root cause, and resolution rather than omitted.

---

## Directory Structure

```
aws/
├── automation-iac/              # EC2 bootstrapping, SNS/SQS event routing, S3 replication with DynamoDB auditing
├── compute/                     # EC2 provisioning, ALB, ASG, EBS, AMI, right-sizing, access controls
├── containers/                  # ECR, ECS Fargate, EKS cluster provisioning and hardening
├── databases/                   # RDS MySQL, DynamoDB, snapshot/restore, EC2-to-RDS connectivity
├── identity-and-security/       # IAM users, groups, roles, policies, instance profiles, KMS encryption
├── monitoring-and-logging/      # Cross-VPC log ingestion pipeline, CloudWatch alarms, SNS alerting
├── networking/                  # VPC design, ALB, NAT Gateway, NAT Instance, VPC Peering, Security Groups
├── serverless/                  # Lambda lifecycle across CLI, CloudFormation, and Console provisioning
├── storage/                     # S3 static hosting, versioning, bucket migration, EBS expansion, snapshots
├── troubleshooting/             # Production incident investigations with structured root cause analysis
└── README.md
```

---

## Domain Summaries

---

### [Automation and IaC](./automation-iac/)

Three infrastructure projects built exclusively through AWS CLI and CloudFormation, each adding a layer: single resource provisioning, event-driven message routing, and a fully auditable serverless pipeline.

| Project | Summary |
|---|---|
| [EC2 Nginx Bootstrap](./automation-iac/ec2-nginx-bootstrap/) | Idempotent user-data provisioning with dynamic AMI resolution and SSM Session Manager access. No SSH key pair, no hardcoded image IDs. |
| [SNS Priority Queue Orchestration](./automation-iac/event-driven-priority-queue-orchestration/) | SNS attribute-based routing to SQS with a Lambda poll-with-fallback pattern. Five failures documented with exact error output and resolution, including a CLI v1/v2 incompatibility and IAM 403 rollback. |
| [S3 Replication with DynamoDB Auditing](./automation-iac/serverless-cross-bucket-replication-with-metadata-auditing/) | Event-triggered Lambda copies objects across buckets and writes a structured DynamoDB audit record on every invocation, including on failure. Confused deputy protection enforced at permission binding. |

---

### [Compute](./compute/)

Thirteen EC2 projects covering the full instance lifecycle, high-availability architecture, and operational hardening.

| Project | Summary |
|---|---|
| [High-Availability ALB + ASG](./compute/high-availability-alb-ec2-deployment/) | Multi-AZ Auto Scaling Group with an internet-facing ALB, ELB health checks, and CPU-based TargetTracking. Nginx bootstrapped via User Data on Amazon Linux 2. |
| [Resilient Web Tier: ALB + EC2 Nginx](./compute/resilient-web-tier-architecture/) | Source-group-based SG chaining eliminates direct internet exposure to EC2. Documents a real AZ mismatch failure, its root cause, and the remediation path. |
| [EC2 CLI Launch Basics](./compute/ec2-cli-launch-basics/) | Dynamic AMI resolution, key pair creation, and instance lifecycle validation. No hardcoded IDs anywhere in the workflow. |
| [Elastic IP Setup](./compute/ec2-elastic-ip-setup/) | EIP allocation and association with CLI-based validation across both `describe-addresses` and `describe-instances`. |
| [Secure SSH Access](./compute/ec2-ssh-access/) | RSA key injection and root login configured entirely via User Data. Security implications documented with the SSM alternative. |
| [EBS Volume Attachment](./compute/ebs-volume-attachment/) | Tag-based resource resolution and hot attachment at `/dev/sdb` without instance restart. |
| [EC2 Storage Expansion](./compute/ec2-storage-expansion/) | `gp3` volume provisioning with pre-attachment state verification. |
| [AMI Creation](./compute/ec2-ami-create/) | Point-in-time image capture with status monitoring through to `available`. |
| [Instance Right-Sizing](./compute/ec2-instance-resize/) | Downgrade from `t2.micro` to `t2.nano` with a stop, modify, and restart cycle. Production FinOps pattern. |
| [Stop Protection](./compute/ec2-stop-protection/) | Console-applied stop protection to block accidental state changes. |
| [Termination Protection](./compute/ec2-termination-protection/) | Console-applied termination protection on a named production instance. |
| [Key Pair Creation](./compute/ec2-launch-secure/) | CLI key generation with `--query 'KeyMaterial'` piped directly to a `.pem` file and `chmod 400` applied. |
| [EC2 Cleanup](./compute/ec2-cleanup/) | Controlled instance termination with confirmed `terminated` state. |

---

### [Containers](./containers/)

Private registry, serverless container deployment, and Kubernetes control plane provisioning, all via AWS CLI with no `eksctl` dependency.

| Project | Summary |
|---|---|
| [Container Registry (ECR)](./containers/container-registry-ecr/) | Private ECR repository with token-based Docker authentication and post-push digest verification. |
| [ECS Fargate Deployment](./containers/ecr-ecs-fargate-deployment/) | Full pipeline from Dockerfile to publicly reachable Nginx service. IAM role creation, `awsvpc` networking, CloudWatch log pre-provisioning, and HTTP 200 validation. |
| [EKS Cluster Provisioning](./containers/eks-cluster-provisioning-and-hardening/) | Private EKS cluster on Kubernetes v1.35. Public endpoint disabled. All three Auto Mode components disabled in a single atomic API call to satisfy EKS validation constraints. |

---

### [Databases](./databases/)

Managed database provisioning, private networking, backup and recovery, and NoSQL table operations.

| Project | Summary |
|---|---|
| [DynamoDB Table Provisioning](./databases/dynamodb-table-provisioning-seeding/) | `PAY_PER_REQUEST` table with mandatory `wait table-exists` gate before writes and JMESPath-based targeted verification. |
| [RDS MySQL Scalable Provisioning](./databases/rds-mysql-scalable-provisioning/) | Private MySQL 8.4 instance with storage autoscaling. Two CLI v1 errors documented: boolean flag syntax and password character rejection at the API layer. |
| [EC2-to-RDS Private Connectivity](./databases/rds-private-mysql-ec2-connectivity/) | Security group chaining scopes port 3306 to the application tier only. `sed`-based runtime credential injection. Browser-confirmed end-to-end connection. |
| [RDS Snapshot and Restore](./databases/rds-snapshot-restore-ops/) | Full snapshot-to-restore pipeline with state transition monitoring documented as expected behavior, not failure. |

---

### [Identity and Security](./identity-and-security/)

IAM primitives, instance profile-based authentication, and KMS symmetric encryption with cryptographic integrity verification.

| Project | Summary |
|---|---|
| [EC2-to-S3 IAM Integration](./identity-and-security/ec2-s3-iam-integration/) | Credential-free S3 access via IMDS instance profile. Least-privilege policy scoped to specific bucket ARN at both bucket and object levels. |
| [KMS Encryption Workflow](./identity-and-security/kms-data-encryption-workflow/) | Full encrypt-decrypt cycle with binary-safe ciphertext storage. Resolves a critical base64 encoding mismatch that causes `InvalidCiphertextException` at decryption. MD5 checksum and `diff` verify round-trip integrity. |
| [IAM Group Creation](./identity-and-security/iam-group-creation/) | Group provisioning with zero-membership state verified as a clean baseline. |
| [IAM Read-Only EC2 Policy](./identity-and-security/iam-readonly-ec2-policy/) | Custom policy scoped to `DescribeInstances`, `DescribeImages`, and `DescribeSnapshots`. No reliance on AWS-managed policies. |
| [IAM Role for EC2](./identity-and-security/iam-roles-ec2/) | Trust policy restricted to `ec2.amazonaws.com` only. Dynamic ARN resolution makes the pattern CI/CD-portable. |
| [IAM User Creation](./identity-and-security/iam-user-creation/) | Minimal provisioning footprint: user created, no keys or policies attached until required. |
| [IAM User Policy Attachment](./identity-and-security/iam-user-policy-attachment/) | Four-phase verify, locate, attach, confirm workflow with dynamic ARN resolution at each step. |

---

### [Monitoring and Logging](./monitoring-and-logging/)

Automated log ingestion pipeline and threshold-based alerting, both implemented without manual intervention after initial setup.

| Project | Summary |
|---|---|
| [Cross-VPC Log Ingestion Pipeline](./monitoring-and-logging/cross-vpc-log-ingestion-and-s3-archival-pipeline/) | Dual-cron pipeline ships boot logs from a private, internet-isolated EC2 instance across VPC Peering to a relay instance, then to a private S3 bucket via IAM instance profile. No NAT Gateway, no static credentials. Absolute binary paths in cron prevent silent PATH failures. |
| [EC2 Performance Threshold Alerting](./monitoring-and-logging/ec2-performance-threshold-alerting/) | CloudWatch alarm at 90% CPU with SNS fan-out. Single 5-minute evaluation period suitable for burst-sensitive workloads. Full validation via `describe-alarms` and `describe-instance-status`. |

---

### [Networking](./networking/)

Eight projects spanning VPC design, internet ingress, private egress, cross-VPC connectivity, and access control, all matched against production architecture patterns.

| Project | Summary |
|---|---|
| [ALB + EC2 + Nginx](./networking/application-load-balancer-ec2-nginx/) | SG-to-SG chaining shields EC2 from direct internet access. Multi-subnet ALB placement for availability. |
| [Public Infrastructure Baseline](./networking/aws-public-infrastructure-baseline/) | Non-default VPC with IGW, route table, and public subnet. DNS support and hostnames enabled at creation. |
| [EC2 ENI Attachment](./networking/ec2-eni-attachment/) | Hot ENI attachment at device index 1 with `in-use` status confirmed without instance restart. |
| [VPC Peering](./networking/inter-vpc-connectivity/) | Bidirectional peering between `172.31.0.0/16` and `10.1.0.0/16`. Four sequential blockers resolved: peering state, route tables, security groups, and SSH key trust. Confirmed 0% packet loss. |
| [NAT Gateway Egress](./networking/nat-gateway-private-subnet-egress/) | Managed NAT Gateway in the same AZ as the private subnet. `wait nat-gateway-available` prevents race conditions. S3 upload used as the true end-to-end validation. |
| [NAT Instance Egress](./networking/private-subnet-egress-via-nat-instance/) | `iptables MASQUERADE` with `ip_forward` on Amazon Linux 2. `iptables-save` persistence pattern documented. NAT Instance vs. NAT Gateway trade-off explicitly analyzed. |
| [Security Groups and NACLs](./networking/security-groups-nacl/) | Least-privilege inbound rules scoped to required ports with a production note on SSH restriction. |
| [Subnet Creation](./networking/subnet-creation-default-vpc/) | Pre-creation CIDR audit to prevent overlap. Tagging applied immediately post-creation. |

---

### [Serverless](./serverless/)

Lambda provisioned three ways: CLI, CloudFormation, and Console. Each targets the same functional outcome, demonstrating deployment-model flexibility and documenting toolchain-specific failure modes.

| Project | Summary |
|---|---|
| [Lambda CLI Deployment](./serverless/lambda-cli-deployment/) | `fileb://` binary upload, dynamic role ARN resolution, and `Active` state polling before invocation. CI/CD-ready pattern. |
| [Lambda CloudFormation Stack](./serverless/lambda-cloudformation-deployment-stack/) | Stack-managed IAM role with `CAPABILITY_NAMED_IAM`. `wait stack-create-complete` used as a blocking gate. Inline log decoding via `base64 --decode` eliminates console dependency. |
| [Lambda Console Deployment](./serverless/lambda-greeting-function-deployment/) | Console-driven workflow with `json.dumps()` response serialization. Region selector drift and Deploy vs. Save distinction documented. |

---

### [Storage](./storage/)

Full lifecycle of S3 and EBS storage management: hosting, versioning, migration, live volume expansion, and snapshots.

| Project | Summary |
|---|---|
| [S3 Static Website Hosting](./storage/s3-static-website-hosting/) | Six-phase provisioning from bucket creation to `HTTP 200` validation. Explicit `--content-type` on upload prevents download-trigger behavior. |
| [EC2 Root Volume Expansion](./storage/ec2-root-volume-expansion-and-filesystem-resize/) | Three-layer expansion: AWS block device, `growpart` partition resize, `xfs_growfs` online filesystem growth. Zero downtime, no reboot. GPT PMBR mismatch documented as expected. |
| [S3 Bucket Migration](./storage/s3-bucket-migration-sync/) | `aws s3 sync` migration with `--dryrun` re-run used as a consistency gate rather than a manual object count. |
| [S3 Versioning](./storage/s3-versioning/) | Three-command workflow: baseline state check, versioning apply, post-change confirmation. |
| [EBS Snapshots](./storage/ebs-snapshots/) | Named snapshot with `completed` status explicitly verified before close. Provides a validated point-in-time recovery baseline. |

---

### [Troubleshooting](./troubleshooting/)

Production incident investigations with structured diagnosis, root cause identification, and verified resolution.

| Project | Summary |
|---|---|
| [Internet Gateway Routing Remediation](./troubleshooting/internet-gateway-routing-remediation/) | Restored public HTTP access to a running Nginx instance. Layer-by-layer diagnosis: IGW attachment, route table coverage, `MapPublicIpOnLaunch` attribute, and security group ingress. Two misconfigurations resolved; SSH unblocked after SSM confirmed unavailable. HTTP 200 confirmed via `curl`. |

---

## Technologies and Tools

| Category | Technologies |
|---|---|
| Cloud Provider | AWS (us-east-1) |
| Infrastructure as Code | AWS CloudFormation (YAML) |
| Compute | EC2 (Amazon Linux 2, Ubuntu 22.04 LTS, t2.micro/nano), AWS Lambda (Python 3.12), Auto Scaling Groups |
| Containers | Amazon ECR, Amazon ECS Fargate, Amazon EKS (Kubernetes v1.35) |
| Networking | VPC, Subnets, Internet Gateway, NAT Gateway, NAT Instance, Route Tables, VPC Peering, ALB, ENI, Elastic IP, Security Groups, NACLs |
| Databases | Amazon RDS (MySQL 8.4), Amazon DynamoDB (PAY_PER_REQUEST) |
| Messaging | Amazon SNS (attribute-based filter policies), Amazon SQS (standard queues, long-polling) |
| Storage | Amazon S3 (bucket policies, event notifications, static hosting, versioning), Amazon EBS (gp3) |
| Identity and Access | AWS IAM (users, groups, roles, managed policies, instance profiles, trust policies), AWS KMS (CMK, aliases, symmetric encryption) |
| Observability | Amazon CloudWatch (metrics, alarms, logs), Amazon SNS |
| Scripting and Tooling | AWS CLI v1 and v2, Bash, Python 3.12, `sed`, `curl`, `growpart`, `xfs_growfs`, `iptables`, `base64`, `jq`-style JMESPath |
| Web Server | Nginx (Amazon Linux 2 via `amazon-linux-extras`, Ubuntu via `apt`) |
| Access Patterns | EC2 Instance Connect, AWS Systems Manager Session Manager, SSH |

---

## Key Skills Demonstrated

**CLI-First Infrastructure Automation**
All resources provisioned, verified, and decommissioned through AWS CLI or CloudFormation. No console-only steps in the critical path. Patterns are directly scriptable for CI/CD integration.

**Least-Privilege IAM Design**
Policies scoped at the action, resource, and principal level throughout. Trust policies restricted to named service principals. Instance profiles used instead of static credentials on EC2 workloads. `iam:AttachRolePolicy` vs. `iam:PutRolePolicy` distinction resolved from a real 403 failure.

**Production Networking Patterns**
SG-to-SG chaining for ALB-to-EC2 isolation. NAT Gateway vs. NAT Instance trade-off with documented cost and reliability implications. VPC Peering with bidirectional route table configuration. Multi-layer connectivity debugging across peering state, routing, security groups, and SSH trust.

**Event-Driven and Serverless Architecture**
SNS attribute-based message routing to SQS. Lambda poll-with-fallback priority queue processing. S3-triggered replication with unconditional DynamoDB audit logging including failure paths. Confused deputy protection on Lambda permissions.

**Structured Troubleshooting and Error Documentation**
Failures documented with exact error output, root cause, and resolution rather than omitted. Covers CloudFormation rollback conditions, CLI version incompatibilities, binary encoding mismatches, AZ placement failures, partition table warnings, and IAM eventual consistency.

**Operational Discipline**
Pre-flight identity and region verification before every provisioning sequence. `aws ec2 wait` and `aws cloudformation wait` used as mandatory execution gates. Validation at the application layer (HTTP 200, S3 upload, ICMP ping) rather than infrastructure state alone.

---

## How to Navigate

Each subdirectory contains a domain-level `README.md` with project summaries, and each project subdirectory contains a self-contained `README.md` with:

- Architecture overview or system diagram
- Full CLI command sequences with expected outputs
- Screenshots at each validation checkpoint
- Errors encountered with root cause analysis and resolution
- Lessons learned and best practices

**Recommended entry points by interest:**

| Goal | Start here |
|---|---|
| End-to-end IaC automation | [`automation-iac/event-driven-priority-queue-orchestration`](./automation-iac/event-driven-priority-queue-orchestration/) |
| Multi-component compute architecture | [`compute/high-availability-alb-ec2-deployment`](./compute/high-availability-alb-ec2-deployment/) |
| Serverless container deployment | [`containers/ecr-ecs-fargate-deployment`](./containers/ecr-ecs-fargate-deployment/) |
| IAM and encryption depth | [`identity-and-security/kms-data-encryption-workflow`](./identity-and-security/kms-data-encryption-workflow/) |
| Cross-VPC networking and incident resolution | [`troubleshooting/internet-gateway-routing-remediation`](./troubleshooting/internet-gateway-routing-remediation/) |
| Private subnet egress trade-off analysis | [`networking/private-subnet-egress-via-nat-instance`](./networking/private-subnet-egress-via-nat-instance/) |
| Audit-compliant observability pipeline | [`automation-iac/serverless-cross-bucket-replication-with-metadata-auditing`](./automation-iac/serverless-cross-bucket-replication-with-metadata-auditing/) |

> All projects provisioned in `us-east-1`. Run the Cleanup commands in each project README after reproducing to avoid unnecessary charges.

---

**Author:** Arinze Edeh | Cloud Infrastructure and DevOps Automation | [GitHub: arinze-edeh](https://github.com/arinze-edeh)
