# automation-iac

**Cloud Infrastructure Automation | AWS CLI | DevOps | IaC**

![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=flat-square&logo=amazon-aws)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=flat-square&logo=gnu-bash)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python)
![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-FF9900?style=flat-square)
![Region](https://img.shields.io/badge/Region-us--east--1-blue?style=flat-square)

---

## Overview

A collection of AWS infrastructure automation projects built entirely through the CLI and CloudFormation, with no console interaction at any stage. Each project is self-contained, fully documented, and structured around patterns common in scalable SaaS platforms, internal DevOps tooling, and cloud-native systems.

These projects simulate production-style infrastructure scenarios including web service provisioning, event-driven message processing, and audit-compliant data pipelines. They reflect the engineering standards expected in environments where reproducibility, security, and observability are non-negotiable.

**Consistent practices across all projects:**

- All resources provisioned, verified, and decommissioned through CLI or IaC workflows
- Least-privilege IAM applied at every layer
- Every deployment phase confirmed with read-back verification commands
- Structured troubleshooting with documented error resolution paths

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

## Project Summaries

---

### [1. Automated EC2 Nginx Web Server Provisioning via AWS CLI](./ec2-nginx-bootstrap/README.md)

**Directory:** [`ec2-nginx-bootstrap/`](./ec2-nginx-bootstrap/)

> **At a Glance**
> | | |
> |---|---|
> | **What it does** | Provisions a public-facing Ubuntu EC2 instance running Nginx from zero using only the AWS CLI |
> | **Key services** | EC2, VPC, Security Groups, SSM Parameter Store, Nginx |
> | **Core concept** | Idempotent user-data bootstrapping with tag-based resource querying and zero SSH exposure |

**Purpose**

Establishes a repeatable, auditable web server deployment pattern suitable for scripted or pipeline-driven environments, without any manual console steps.

**Approach**

Validated IAM identity using `aws sts get-caller-identity` before touching any resources. Created a dedicated security group scoped to TCP port 80 only. Retrieved the latest Ubuntu 22.04 LTS AMI dynamically from SSM Parameter Store to avoid hardcoding deprecated image IDs.

Launched the instance tagged `Name=xfusion-ec2` for deterministic downstream querying. A Bash user data script handles Nginx installation, service start, and `systemctl enable` at first boot. Deployment validated with `curl -I`, confirming HTTP 200 and the `Server: nginx` header.

**Key Design Decisions**

- No SSH key pair attached; administrative access routes through AWS Systems Manager Session Manager, eliminating the need for inbound port 22
- User data script is idempotent: safe to re-execute on any fresh instance without side effects

**Outcome**

Fully operational public Nginx web server deployed end-to-end through CLI commands. Workflow is directly scriptable for CI/CD integration.

**Tools and Services**

AWS CLI, EC2, VPC, Security Groups, Ubuntu 22.04 LTS, Nginx, Bash, AWS Systems Manager Parameter Store

---

### [2. AWS Event-Driven Priority Queuing: SNS Attribute-Based Routing, Lambda Long-Poll Fallback, and CloudFormation IaC Troubleshooting](./event-driven-priority-queue-orchestration/README.md)

**Directory:** [`event-driven-priority-queue-orchestration/`](./event-driven-priority-queue-orchestration/)

> **At a Glance**
> | | |
> |---|---|
> | **What it does** | Routes messages to separate SQS queues by priority using SNS filter policies, processed by a Lambda function that always drains high-priority first |
> | **Key services** | CloudFormation, SNS, SQS, Lambda, IAM |
> | **Core concept** | Attribute-based SNS routing combined with a poll-with-fallback Lambda pattern, fully provisioned as IaC with a documented error registry |

**Purpose**

Implements a serverless priority queuing system guaranteeing high-priority messages are always processed before low-priority ones. All infrastructure provisioned via a single CloudFormation template.

**Approach**

The CloudFormation template defines two SQS queues, one SNS topic with attribute-based filter policies routing on the `priority` message attribute, a Python 3.12 Lambda function, and a least-privilege IAM role.

A standalone `AWS::IAM::ManagedPolicy` resource replaces the inline `Policies:` block on the IAM role. This is a critical distinction: inline policies require `iam:PutRolePolicy`, while managed policies require only `iam:AttachRolePolicy`, which resolves a real 403 deployment failure documented in the error registry.

Lambda reads queue URLs from environment variables injected by CloudFormation intrinsic functions, keeping configuration fully decoupled from code. SQS long-polling (`WaitTimeSeconds=3`) is enabled on both queues to reduce empty-receive API calls.

**Documented Errors and Resolutions**

Five distinct failures are captured with full root cause analysis:

- Python `yaml.safe_load` rejecting CloudFormation intrinsic tags (`!Ref`, `!GetAtt`)
- CloudFormation rollback caused by `iam:PutRolePolicy` 403 on the lab IAM principal
- AWS CLI v1 rejecting `--cli-binary-format`, a v2-only flag, across all four invocations
- Lambda timeout at 3 seconds undersized against a worst-case 6-second two-queue long-poll path

**Outcome**

Four test messages (two high, two low) published and consumed in strict priority order across five Lambda invocations. Full error registry delivered alongside working infrastructure.

**Tools and Services**

AWS CloudFormation, SQS, SNS, Lambda (Python 3.12), IAM, AWS CLI v1, CloudWatch Logs

---

### [3. Automated S3-to-S3 File Replication Pipeline with Lambda and DynamoDB Audit Logging](./serverless-cross-bucket-replication-with-metadata-auditing/README.md)

**Directory:** [`serverless-cross-bucket-replication-with-metadata-auditing/`](./serverless-cross-bucket-replication-with-metadata-auditing/)

> **At a Glance**
> | | |
> |---|---|
> | **What it does** | Automatically replicates objects from a public intake S3 bucket to a private destination bucket, writing a structured audit record to DynamoDB on every event |
> | **Key services** | S3, Lambda, DynamoDB, IAM, CloudWatch Logs |
> | **Core concept** | Event-driven serverless pipeline with least-privilege IAM, confused deputy protection, and tamper-evident audit logging |

**Purpose**

Automates file replication between storage tiers with a durable, structured audit trail on every transfer. Addresses a pattern common in data ingestion pipelines and compliance-sensitive environments where file movement must be logged and verifiable.

**Approach**

Two S3 buckets are provisioned with opposite access profiles: the intake bucket has public read enabled via bucket policy; the private bucket has all four public access block flags set to `true`.

The Lambda execution role's custom policy grants only `s3:GetObject` on the source bucket, `s3:PutObject` on the destination, and `dynamodb:PutItem` on the specific table ARN. No wildcard resource statements are used. The `lambda add-permission` call includes both `--source-arn` and `--source-account` to prevent the confused deputy vulnerability. The S3 event notification is applied only after this permission is in place, as S3 validates it at configuration time.

The Lambda function handles four responsibilities on each invocation:

- Parses the S3 event record to extract bucket and object key
- Copies the object to the private destination bucket
- Builds a DynamoDB audit record with a UUID partition key, UTC timestamp, source and destination buckets, object key, and status
- Uses a nested try/catch to write a `Failure` record even when the copy operation fails, preserving audit completeness

Environment-specific resource names are injected at deploy time via `sed` substitution on named placeholders, enabling the same function template to target any environment without source code changes. A 10-second `sleep` between IAM role creation and Lambda deployment accounts for IAM eventual consistency propagation. DynamoDB uses `PAY_PER_REQUEST` billing to match the bursty, event-driven write pattern without provisioned capacity management.

**Outcome**

Uploading `sample.zip` triggered the Lambda function, replicated the file to the private bucket within 15 seconds, and produced a correctly structured DynamoDB audit record with `"Status": "Success"`. CloudWatch logs confirmed all execution stages completed cleanly, validated using `aws logs filter-log-events`.

**Tools and Services**

AWS CLI, S3, Lambda (Python 3.12), DynamoDB, IAM, CloudWatch Logs, Bash

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
| Observability | Amazon CloudWatch Logs, AWS CloudTrail (recommended) |
| Scripting and Tooling | AWS CLI v1 and v2, Bash, Python 3.12, `sed`, `curl` |
| Web Server | Nginx (systemd-managed, user-data bootstrapped) |

---

## Key Outcomes and Skills Demonstrated

**Infrastructure as Code and CLI Automation**

All three projects provision, configure, and validate AWS infrastructure exclusively through CLI commands and CloudFormation templates. Every workflow is directly scriptable and suitable for CI/CD pipeline integration.

**Least-Privilege IAM Design**

Least-privilege IAM is applied at every layer: security group ingress locked to required ports only, Lambda execution roles scoped to specific resource ARNs, and a practical understanding of the difference between `iam:PutRolePolicy` and `iam:AttachRolePolicy` drawn from a real deployment failure and its resolution.

**Event-Driven Architecture**

Two projects implement distinct event-driven patterns. The priority queue system uses SNS attribute-based routing to SQS with a Lambda poll-with-fallback handler. The replication pipeline uses S3 event notifications to trigger Lambda with a resource-based invocation policy and confused deputy mitigation.

**Structured Troubleshooting**

The priority queue project captures five failure modes with exact error output, root cause analysis, and resolution steps. Diagnostic methodology includes filtering `describe-stack-events` to `CREATE_FAILED` events, identifying CLI version incompatibilities, and sizing Lambda timeouts against worst-case execution paths.

**Operational Verification**

No deployment is considered complete until state is confirmed through read-back CLI queries (`get-function`, `get-bucket-notification-configuration`, `describe-instances`, `dynamodb scan`), HTTP response validation, and CloudWatch log inspection.

**Audit-Compliant Observability**

The S3 replication pipeline writes structured DynamoDB audit records with UUID partition keys, UTC timestamps, and explicit `Failure` entries on error. Log output is validated at the execution level through CloudWatch, not just at the HTTP layer.

---

## How to Navigate This Repository

Each project directory contains a self-contained `README.md` structured as follows:

- **Problem Statement**: The operational need and what manual process it replaces
- **Architecture**: A text diagram of the resource topology and data flow
- **Prerequisites**: Required tools, permissions, and environment configuration
- **Step-by-Step Implementation**: Every CLI command used, with expected output and annotated screenshots
- **Error Registry** (where applicable): Full error output, root cause, and resolution for every failure encountered
- **Validation Checklist**: A table confirming the expected state of every provisioned resource
- **Best Practices**: Design decisions and operational guidance derived from the implementation
- **Cleanup**: Commands to terminate and delete all provisioned resources

To reproduce any project: clone the repository, confirm your identity with `aws sts get-caller-identity`, verify your region with `aws configure get region`, and follow the step-by-step commands in the relevant project README.

---

## Author

**Arinze Edeh**
Cloud Infrastructure and DevOps Automation
[GitHub: arinze-edeh](https://github.com/arinze-edeh)

---

*All projects provisioned in AWS region `us-east-1`. Refer to the Cleanup section in each project README for teardown commands to avoid unnecessary charges.*
