# automation-iac

**Cloud Infrastructure Automation | AWS CLI | DevOps | IaC**

![AWS](https://img.shields.io/badge/AWS-Cloud-FF9900?style=flat-square&logo=amazon-aws)
![Bash](https://img.shields.io/badge/Bash-Scripting-4EAA25?style=flat-square&logo=gnu-bash)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python)
![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-FF9900?style=flat-square)
![Region](https://img.shields.io/badge/Region-us--east--1-blue?style=flat-square)

---

## Overview

This repository documents a collection of hands-on AWS infrastructure automation projects completed using the AWS CLI, CloudFormation, and serverless services. Each project demonstrates a self-contained, production-oriented workflow covering provisioning, configuration, IAM scoping, event-driven architecture, and operational verification.

The work in this directory reflects principles expected in professional DevOps environments: reproducible deployments driven entirely by code, least-privilege IAM design, structured audit trails, observable infrastructure, and systematic troubleshooting with documented resolution paths.

All projects were executed without reliance on the AWS Management Console. Every resource was provisioned, verified, and decommissioned exclusively through CLI-driven or IaC-driven workflows, making each implementation suitable as a reference for CI/CD pipeline integration.

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
    └── README.md 
```

---

## Project Summaries

---

### 1. Automated EC2 Nginx Web Server Provisioning via AWS CLI

**Directory:** `ec2-nginx-bootstrap/`

**Purpose**

Provision a publicly accessible Ubuntu EC2 instance running Nginx, using only the AWS CLI, without any manual console interaction. The project establishes a repeatable, auditable deployment pattern suitable for web server bootstrapping in scripted or pipeline-driven environments.

**Approach**

The workflow begins with IAM identity validation via `aws sts get-caller-identity` to confirm the correct principal and account before any resource creation. A dedicated security group (`xfusion-web-sg`) is created and scoped to permit only TCP port 80 inbound traffic, following the principle of least privilege. A Bash user data script installs Nginx, starts the service, and enables it for automatic restarts on reboot. The EC2 instance is launched with the `t2.micro` instance type using the latest Ubuntu 22.04 LTS AMI retrieved dynamically from AWS Systems Manager Parameter Store to avoid hardcoding deprecated image IDs. The instance is tagged at launch (`Name=xfusion-ec2`) to enable deterministic tag-based resource querying in all downstream steps. Deployment is validated with `curl -I` against the public IP, confirming an HTTP 200 response and the `Server: nginx` header.

**Key Design Decisions**

No SSH key pair is attached to the instance. Administrative access, if required, is designed to route through AWS Systems Manager Session Manager, eliminating the need for inbound port 22 and reducing the attack surface. The user data bootstrap script is idempotent: `apt-get install -y` is non-destructive if Nginx is already present, and `systemctl enable` produces the same result on repeated executions.

**Outcome**

A fully operational, publicly accessible Nginx web server deployed end-to-end through CLI commands with zero console interaction. The workflow is directly scriptable for use in CI/CD pipelines.

**Tools and Services**

AWS CLI, EC2, VPC, Security Groups, Ubuntu 22.04 LTS, Nginx, Bash, AWS Systems Manager Parameter Store

---

### 2. AWS Event-Driven Priority Queuing: SNS Attribute-Based Routing, Lambda Long-Poll Fallback, and CloudFormation IaC Troubleshooting

**Directory:** `event-driven-priority-queue-orchestration/`

**Purpose**

Design and deploy a serverless priority message routing system that guarantees high-priority messages are always processed before low-priority messages, using SNS filter policies to route to separate SQS queues and a Lambda function implementing a poll-with-fallback pattern. All infrastructure is provisioned via a single CloudFormation template.

**Approach**

The CloudFormation template defines two SQS queues (`nautilus-High-Priority-Queue` and `nautilus-Low-Priority-Queue`), one SNS topic (`nautilus-Priority-Queues-Topic`) with attribute-based filter policies routing on the `priority` message attribute, a Python 3.12 Lambda function implementing the poll-with-fallback logic, and a least-privilege IAM role. A standalone `AWS::IAM::ManagedPolicy` resource is used instead of an inline `Policies:` block on the IAM role, which is a critical distinction in environments where `iam:PutRolePolicy` is not permitted but `iam:AttachRolePolicy` is. The Lambda function reads queue URLs from environment variables injected by CloudFormation intrinsic functions, keeping configuration decoupled from code. SQS long-polling (`WaitTimeSeconds=3`) is enabled to reduce empty-receive API calls and lower cost.

The project documents five distinct errors encountered during implementation, each with full root cause analysis and resolution: Python YAML validation failing on CloudFormation intrinsic tags, a CloudFormation rollback traced to an IAM permission gap, AWS CLI v1 vs. v2 flag incompatibility on Lambda invocations, and a Lambda timeout failure caused by undersizing the timeout against the worst-case two-queue long-poll execution path.

**Priority Routing Logic**

```
Lambda invoked
      |
      v
Poll high-priority queue (WaitTimeSeconds=3)
      |
      +-- Message found  --> delete --> return result
      |
      +-- Empty after 3s --> poll low-priority queue (WaitTimeSeconds=3)
                                  |
                                  +-- Message found  --> delete --> return result
                                  |
                                  +-- Empty after 3s --> return "No more messages to poll"

Worst-case blocking time:  up to 6 seconds
Configured Lambda timeout: 10 seconds (4-second safety margin)
```

**Outcome**

A fully operational priority queuing system validated by publishing four test messages (two high, two low) and confirming they were consumed in strict priority order across five Lambda invocations. All infrastructure was provisioned through CloudFormation with a complete error registry documenting every failure and resolution encountered during deployment.

**Tools and Services**

AWS CloudFormation, SQS, SNS, Lambda (Python 3.12), IAM, AWS CLI v1, CloudWatch Logs

---

### 3. Automated S3-to-S3 File Replication Pipeline with Lambda and DynamoDB Audit Logging

**Directory:** `serverless-cross-bucket-replication-with-metadata-auditing/`

**Purpose**

Implement a fully event-driven, serverless file replication pipeline that automatically copies objects uploaded to a public intake S3 bucket into a private destination bucket, writing a structured audit log entry to DynamoDB on every successful or failed copy event.

**Approach**

Two S3 buckets are provisioned with explicitly opposite access configurations: the intake bucket (`datacenter-public-14515`) has public read access enabled via a bucket policy, while the private bucket (`datacenter-private-1666`) has all four public access block flags set to `true`. A Lambda execution IAM role is created with a custom scoped policy granting only `s3:GetObject` on the source bucket, `s3:PutObject` on the destination bucket, and `dynamodb:PutItem` on the specific audit table ARN, with no wildcard resource statements. The Lambda function is granted invocation permission via `lambda add-permission` with both `--source-arn` and `--source-account` specified to prevent the confused deputy vulnerability. The S3 event notification is configured after the Lambda permission is in place, as S3 validates the invocation permission at configuration time.

The Lambda function parses the S3 event record, copies the object, builds a structured DynamoDB audit record with a UUID partition key, source and destination buckets, object key, UTC timestamp, and status, and persists the record. A nested try/catch pattern attempts to write a `Failure` audit entry even when the copy operation fails, preserving audit completeness. Environment-specific resource names are injected at deploy time using `sed` substitution on placeholder values in the function source, enabling the same template to be reused across environments.

A 10-second `sleep` is inserted between IAM role creation and Lambda function deployment to account for IAM eventual consistency propagation, preventing intermittent `InvalidParameterValueException` failures. The DynamoDB table uses `PAY_PER_REQUEST` billing to match the bursty, event-driven write pattern without provisioned capacity.

**Outcome**

End-to-end validation confirmed that uploading `sample.zip` to the public bucket triggered the Lambda function, produced the replicated file in the private bucket within 15 seconds, and generated a correctly structured DynamoDB audit record with `"Status": "Success"`. CloudWatch log output was validated using `aws logs filter-log-events` to confirm all execution stages completed without errors.

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
| Storage | Amazon S3 (bucket policies, event notifications, public access controls), Amazon DynamoDB (PAY_PER_REQUEST, UUID partition keys) |
| Identity and Access | AWS IAM (roles, managed policies, resource-based policies, least-privilege scoping) |
| Observability | Amazon CloudWatch Logs, AWS CloudTrail (recommended) |
| Scripting and Tooling | AWS CLI v1 and v2, Bash, Python 3.12, `sed`, `curl` |
| Web Server | Nginx (systemd-managed, user-data bootstrapped) |

---

## Key Outcomes and Skills Demonstrated

**Infrastructure as Code and CLI Automation**

All three projects provision, configure, and validate AWS infrastructure exclusively through CLI commands and CloudFormation templates. No console interaction is required. Each workflow is directly scriptable and suitable for integration into CI/CD pipelines.

**IAM Design and Least-Privilege Enforcement**

Each project applies least-privilege IAM: security group ingress restricted to required ports only, Lambda execution roles scoped to specific resource ARNs, and a documented understanding of the distinction between `iam:PutRolePolicy` (inline policies) and `iam:AttachRolePolicy` (managed policies), which resolves a real deployment failure in the priority queue project.

**Event-Driven Architecture**

The priority queue and S3 replication projects both implement event-driven patterns: SNS-to-SQS routing with attribute-based filtering and Lambda poll-with-fallback logic in the first, and S3 event notifications triggering Lambda with a resource-based invocation policy in the second.

**Structured Troubleshooting and Error Documentation**

The priority queue project documents a complete error registry of five distinct failure modes, each with the exact error output, root cause analysis, and resolution. This reflects a professional diagnostic methodology: using `describe-stack-events` filtered to `CREATE_FAILED` to surface specific resource failures from generic CloudFormation rollback messages, identifying CLI version incompatibilities, and correctly sizing Lambda timeouts against worst-case execution paths rather than the happy path.

**Operational Verification**

Every project includes a verification step at each phase. Deployments are not considered complete until resource state is confirmed through read-back CLI queries (`get-function`, `get-bucket-notification-configuration`, `describe-instances`, `dynamodb scan`), HTTP validation, and CloudWatch log inspection.

**Audit and Observability**

The S3 replication project implements a structured DynamoDB audit log with UUID partition keys, UTC timestamps, source and destination metadata, and explicit failure record writes. CloudWatch log output is validated at the execution level, not just at the HTTP response level.

---

## How to Navigate This Repository

Each project directory contains a self-contained `README.md` with the following structure:

- **Problem Statement**: The operational need the project addresses
- **Architecture**: A text-based diagram of the resource topology and data flow
- **Prerequisites**: Required tools, permissions, and environment configuration
- **Step-by-Step Implementation**: Every CLI command used, with expected output and annotated screenshots
- **Error Registry** (where applicable): Full error output, root cause, and resolution for every failure encountered
- **Validation Checklist**: A verification table confirming the expected state of every provisioned resource
- **Best Practices**: Design decisions and operational guidance derived from the implementation
- **Cleanup**: Commands to terminate and delete all provisioned resources to avoid unnecessary charges

To reproduce any project, clone the repository, verify your AWS identity with `aws sts get-caller-identity`, confirm your region with `aws configure get region`, and follow the step-by-step commands in the corresponding project README.

---

## Author

**Arinze Edeh**
Cloud Infrastructure and DevOps Automation
[GitHub: arinze-edeh](https://github.com/arinze-edeh)

---

*All projects provisioned in AWS region `us-east-1`. Resources should be cleaned up after use to avoid incurring unnecessary charges. Refer to the Cleanup section in each project README for teardown commands.*
