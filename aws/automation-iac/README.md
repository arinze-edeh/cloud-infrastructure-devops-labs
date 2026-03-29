# automation-iac

**Cloud Infrastructure Automation | AWS CLI | DevOps | IaC**

![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=flat-square&logo=amazon-aws)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=flat-square&logo=gnu-bash)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python)
![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-FF9900?style=flat-square)
![Region](https://img.shields.io/badge/Region-us--east--1-blue?style=flat-square)

---

## What This Repository Demonstrates

Three AWS infrastructure projects, each one building on the last: a single compute resource provisioned from scratch, a multi-service event-driven messaging system, and a fully auditable data pipeline. Together they show a complete picture of cloud engineering — provisioning, orchestration, and observability — built exclusively through CLI and IaC with no console interaction at any stage.

**Every project is production-patterned:** least-privilege IAM at every layer, read-back CLI verification at every deployment step, and structured error documentation where failures occurred.

---

## Skills at a Glance

| Area | Demonstrated by |
|---|---|
| CLI & IaC automation | All resources provisioned, verified, and decommissioned through AWS CLI or CloudFormation — zero console steps |
| Least-privilege IAM | Security groups scoped to required ports only; Lambda roles limited to specific resource ARNs; `iam:AttachRolePolicy` vs `iam:PutRolePolicy` distinction resolved from a real 403 failure |
| Event-driven architecture | SNS attribute-based routing to SQS (Project 2); S3 event notifications triggering Lambda (Project 3) |
| Structured troubleshooting | Five failures documented in Project 2 with exact error output, root cause, and resolution — including CLI version incompatibility and Lambda timeout sizing |
| Audit-compliant observability | DynamoDB audit trail with UUID partition keys, UTC timestamps, and `Failure` records on error (Project 3); all deployments validated via CloudWatch log inspection |
| Idempotent bootstrapping | EC2 user-data script safe to re-execute on any fresh instance; no hardcoded AMI IDs |

---

## Directory Structure

```
aws/
└── automation-iac/
    ├── ec2-nginx-bootstrap/
    │   └── README.md
    ├── event-driven-priority-queue-orchestration/
    │   └── README.md
    ├── serverless-cross-bucket-replication-with-metadata-auditing/
    │   └── README.md
    └── README.md  ← (this file)
```

---

## Projects

---

### 1. Automated EC2 Nginx Web Server Provisioning via AWS CLI

**Directory:** [`ec2-nginx-bootstrap/`](./ec2-nginx-bootstrap/)

| | |
|---|---|
| **What it does** | Provisions a public-facing Ubuntu EC2 instance running Nginx from zero using only the AWS CLI |
| **Key services** | EC2, VPC, Security Groups, SSM Parameter Store, Nginx |
| **Core concept** | Idempotent user-data bootstrapping with tag-based resource querying and zero SSH exposure |

**Architecture**

```
Internet
   │  HTTP :80
   ▼
[Security Group] ── TCP 80 only
   │
   ▼
[EC2: xfusion-ec2]
  Ubuntu 22.04 LTS
  Nginx (systemd)
   │
   ▼
[SSM Session Manager] ── admin access (no SSH, no port 22)
```

**Approach**

Validated IAM identity using `aws sts get-caller-identity` before touching any resources. Created a dedicated security group scoped to TCP port 80 only. Retrieved the latest Ubuntu 22.04 LTS AMI dynamically from SSM Parameter Store to avoid hardcoding deprecated image IDs.

Launched the instance tagged `Name=xfusion-ec2` for deterministic downstream querying. A Bash user data script handles Nginx installation, service start, and `systemctl enable` at first boot. Deployment validated with `curl -I`, confirming HTTP 200 and the `Server: nginx` header.

**Key Design Decisions**

- No SSH key pair attached; administrative access routes through AWS Systems Manager Session Manager, eliminating inbound port 22
- User data script is idempotent: safe to re-execute on any fresh instance without side effects
- AMI resolved dynamically from SSM Parameter Store — never a hardcoded, potentially deprecated image ID

**Outcome**

Fully operational public Nginx web server deployed end-to-end through CLI commands. Workflow is directly scriptable for CI/CD integration.

**Tools and Services:** AWS CLI, EC2, VPC, Security Groups, Ubuntu 22.04 LTS, Nginx, Bash, AWS Systems Manager Parameter Store

---

### 2. AWS Event-Driven Priority Queuing: SNS Attribute-Based Routing, Lambda Long-Poll Fallback, and CloudFormation IaC Troubleshooting

**Directory:** [`event-driven-priority-queue-orchestration/`](./event-driven-priority-queue-orchestration/)

| | |
|---|---|
| **What it does** | Routes messages to separate SQS queues by priority using SNS filter policies, processed by a Lambda function that always drains high-priority first |
| **Key services** | CloudFormation, SNS, SQS, Lambda, IAM |
| **Core concept** | Attribute-based SNS routing combined with a poll-with-fallback Lambda pattern, fully provisioned as IaC with a documented error registry |

**Architecture**

```
Publisher
   │
   │  publish(priority="high" | "low")
   ▼
[SNS Topic]
   │              │
   │ filter:      │ filter:
   │ priority=high│ priority=low
   ▼              ▼
[SQS: High]   [SQS: Low]
        \         /
         \       /
          ▼     ▼
        [Lambda Function]
          poll high first
          fallback to low
               │
               ▼
        [CloudWatch Logs]
```

**Approach**

A single CloudFormation template provisions everything: two SQS queues, one SNS topic with `priority` attribute filter policies, a Python 3.12 Lambda function, and an IAM role.

IAM is handled via a standalone `AWS::IAM::ManagedPolicy` rather than an inline `Policies:` block — inline policies require `iam:PutRolePolicy`, managed policies require only `iam:AttachRolePolicy`. This distinction resolved a real 403 rollback failure documented in the error registry.

Queue URLs are injected into Lambda via CloudFormation intrinsic functions, keeping config fully decoupled from code. Long-polling (`WaitTimeSeconds=3`) is enabled on both queues.

**Documented Failures and Resolutions**

| # | Error | Root Cause | Resolution |
|---|---|---|---|
| 1 | `yaml.safe_load` rejects template | CloudFormation intrinsic tags (`!Ref`, `!GetAtt`) are not valid YAML | Use `yaml.load` with `Loader=yaml.FullLoader`, or validate via `aws cloudformation validate-template` |
| 2 | CloudFormation rollback on stack create | `iam:PutRolePolicy` 403 — lab principal lacks permission for inline policies | Replace inline `Policies:` block with a standalone `AWS::IAM::ManagedPolicy` resource |
| 3 | CLI flag rejected on all four invocations | `--cli-binary-format` is AWS CLI v2-only; environment was running v1 | Remove the flag; use `--cli-binary-format raw-in-base64-out` only when v2 is confirmed |
| 4 | Lambda timeout on worst-case path | 3-second timeout undersized against a 6-second two-queue long-poll | Increase timeout to ≥10 seconds to cover both queue polls |
| 5 | Empty receives on SQS during testing | Short-poll returns immediately even when queue is non-empty | Enable long-polling with `WaitTimeSeconds=3` on both queues |

**Outcome**

Four test messages (two high, two low) published and consumed in strict priority order across five Lambda invocations. Full error registry delivered alongside working infrastructure.

**Tools and Services:** AWS CloudFormation, SQS, SNS, Lambda (Python 3.12), IAM, AWS CLI v1, CloudWatch Logs

---

### 3. Automated S3-to-S3 File Replication Pipeline with Lambda and DynamoDB Audit Logging

**Directory:** [`serverless-cross-bucket-replication-with-metadata-auditing/`](./serverless-cross-bucket-replication-with-metadata-auditing/)

| | |
|---|---|
| **What it does** | Automatically replicates objects from a public intake S3 bucket to a private destination bucket, writing a structured audit record to DynamoDB on every event |
| **Key services** | S3, Lambda, DynamoDB, IAM, CloudWatch Logs |
| **Core concept** | Event-driven serverless pipeline with least-privilege IAM, confused deputy protection, and tamper-evident audit logging |

**Architecture**

```
[Client]
   │
   │  s3:PutObject
   ▼
[S3: Intake Bucket]          [IAM Role]
  public-read policy           s3:GetObject (source only)
   │                           s3:PutObject (dest only)
   │  S3 Event Notification    dynamodb:PutItem (table ARN only)
   │  (source-arn + source-        │
   │   account on permission)      │
   ▼                               │
[Lambda Function] ◄────────────────┘
   │         │
   │ copy    │ audit record
   ▼         ▼
[S3: Private Bucket]    [DynamoDB: Audit Table]
  all public access       UUID PK | timestamp
  block = true            source | dest | key | status
                          writes Failure record on error
```

**Approach**

Two S3 buckets with opposite access profiles: the intake bucket has public read via bucket policy; the private bucket has all four public access block flags set to `true`.

The Lambda execution role grants only `s3:GetObject` on the source, `s3:PutObject` on the destination, and `dynamodb:PutItem` on the specific table ARN — no wildcards. The `lambda add-permission` call includes both `--source-arn` and `--source-account` to prevent the confused deputy vulnerability, and is applied before the S3 event notification (S3 validates the permission at configuration time).

On each invocation, Lambda parses the event record, copies the object to the private bucket, and writes a DynamoDB audit record (UUID PK, UTC timestamp, source/dest, object key, status). A nested try/catch ensures a `Failure` record is written even when the copy fails — audit completeness is unconditional.

Resource names are injected at deploy time via `sed` substitution. A 10-second `sleep` after IAM role creation accounts for eventual consistency before Lambda deployment. DynamoDB uses `PAY_PER_REQUEST` to match the bursty write pattern.

**Outcome**

Uploading `sample.zip` triggered the Lambda function, replicated the file to the private bucket within 15 seconds, and produced a correctly structured DynamoDB audit record with `"Status": "Success"`. CloudWatch logs confirmed all execution stages completed cleanly, validated using `aws logs filter-log-events`.

**Tools and Services:** AWS CLI, S3, Lambda (Python 3.12), DynamoDB, IAM, CloudWatch Logs, Bash

---

## Technologies and Tools

| Category | Technologies |
|---|---|
| Cloud Provider | AWS (us-east-1) |
| Infrastructure as Code | AWS CloudFormation |
| Compute | EC2 (Ubuntu 22.04 LTS, t2.micro), AWS Lambda (Python 3.12) |
| Messaging and Queuing | Amazon SQS (standard queues, long-polling), Amazon SNS (attribute-based filter policies) |
| Storage | Amazon S3 (bucket policies, event notifications, access controls), Amazon DynamoDB (PAY_PER_REQUEST, UUID partition keys) |
| Identity and Access | AWS IAM (roles, managed policies, resource-based policies, least-privilege scoping) |
| Observability | Amazon CloudWatch Logs |
| Scripting and Tooling | AWS CLI v1 and v2, Bash, Python 3.12, `sed`, `curl` |
| Web Server | Nginx (systemd-managed, user-data bootstrapped) |

---

## How to Reproduce Any Project

1. Clone the repository
2. Confirm your identity: `aws sts get-caller-identity`
3. Verify your region: `aws configure get region`
4. Follow the step-by-step commands in the relevant project README

Each project README is structured as: **Problem Statement → Architecture → Prerequisites → Step-by-Step Implementation → Error Registry → Validation Checklist → Best Practices → Cleanup**.

> All projects provisioned in `us-east-1`. Run the Cleanup commands in each README after reproducing to avoid unnecessary charges.

---

## Author

**Arinze Edeh** — Cloud Infrastructure and DevOps Automation
[GitHub: arinze-edeh](https://github.com/arinze-edeh)
