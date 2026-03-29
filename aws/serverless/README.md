# AWS Serverless

[![AWS](https://img.shields.io/badge/AWS-Serverless-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](https://aws.amazon.com/serverless/)
[![Lambda](https://img.shields.io/badge/Lambda-Python_3.12-3776AB?style=flat-square&logo=awslambda&logoColor=white)](https://aws.amazon.com/lambda/)
[![IaC](https://img.shields.io/badge/IaC-CloudFormation-orange?style=flat-square&logo=amazonaws)](https://aws.amazon.com/cloudformation/)
[![CLI](https://img.shields.io/badge/Deployment-AWS_CLI-232F3E?style=flat-square&logo=amazonaws)](https://aws.amazon.com/cli/)
[![Region](https://img.shields.io/badge/Region-us--east--1-232F3E?style=flat-square&logo=amazonaws)](https://aws.amazon.com/)

---

## Overview

This directory covers the full AWS Lambda deployment lifecycle across three distinct provisioning methods: direct CLI invocation, CloudFormation IaC stacks, and the AWS Management Console. Each project reflects a different operational context relevant in production environments, ranging from pipeline-ready scriptable deployments to declarative infrastructure-as-code and console-based provisioning workflows.

The work here directly addresses a common production requirement: Lambda functions must be deployable, auditable, and reproducible regardless of the toolchain in use. All three approaches target the same functional outcome (a Python 3.12 handler returning a structured `200` response) but differ in their deployment model, IAM provisioning strategy, and validation method.

---

## Directory Structure

```
aws/serverless/
├── lambda-cli-deployment/               # Lambda provisioned entirely via AWS CLI
├── lambda-cloudformation-deployment-stack/  # Lambda and IAM via CloudFormation IaC
├── lambda-greeting-function-deployment/ # Lambda provisioned via AWS Console
└── README.md                            # This file
```

---

## Project Summaries

---

### [Lambda CLI Deployment](./lambda-cli-deployment/)

**Quick Summary:** Full Lambda lifecycle management using only the AWS CLI. No console, no IaC. Designed for pipeline integration and repeatable scriptable deployments.

| | |
|---|---|
| **Function Name** | `xfusion-lambda-cli` |
| **Runtime** | Python 3.12 |
| **IAM Role** | `lambda_execution_role` (pre-existing) |
| **Region** | `us-east-1` |

**Purpose**
Provision a Lambda function from the command line to validate pipeline-readiness and demonstrate full CLI-based lifecycle management: author, package, role resolve, deploy, and invoke.

**Approach**
- Handler authored via heredoc with a quoted delimiter to prevent shell interpolation
- Deployment package zipped from within the source directory to ensure root-level archive structure, avoiding `Runtime.ImportModuleError`
- IAM role ARN dynamically resolved via `aws iam get-role --query 'Role.Arn'` and captured as a shell variable, eliminating hardcoded ARNs
- Function state polled via `get-function` to confirm `Active` before invocation

**Key Decisions**
- `fileb://` used instead of `file://` for zip upload; `file://` causes base64 encoding and triggers `InvalidParameterValueException`
- ARN dynamically resolved rather than hardcoded to keep the script portable across AWS accounts

**Outcome**
Function deployed and invoked successfully. Response confirmed: `statusCode: 200`, `body: "Welcome to KKE AWS Labs!"`. Pattern is CI/CD-ready with no console dependency.

---

### [Lambda CloudFormation Deployment Stack](./lambda-cloudformation-deployment-stack/)

**Quick Summary:** Lambda function and IAM execution role provisioned as a single CloudFormation stack. All resources are version-controlled, reproducible, and drift-resistant by design.

| | |
|---|---|
| **Stack Name** | `devops-lambda-app` |
| **Function Name** | `devops-lambda` |
| **Runtime** | Python 3.12 |
| **IAM Role** | `lambda_execution_role` (stack-managed) |
| **Region** | `us-east-1` |

**Purpose**
Enforce declarative, auditable Lambda provisioning using CloudFormation as the single source of truth. Eliminate manual console workflows and ensure all resource configuration is captured in version-controlled YAML.

**Approach**
- Template authored at `/root/devops-lambda.yml` with `AWS::IAM::Role` and `AWS::Lambda::Function` as logically dependent resources; CloudFormation resolves provisioning order automatically via `!GetAtt`
- IAM role granted only `AWSLambdaBasicExecutionRole` (least privilege); trust policy scoped exclusively to `lambda.amazonaws.com`
- `CAPABILITY_NAMED_IAM` explicitly passed to acknowledge named role creation and prevent `InsufficientCapabilitiesException`
- `aws cloudformation wait stack-create-complete` used as a blocking gate before resource verification, preventing race conditions in automation
- Inline log decoding via `--log-type Tail` and `base64 --decode` for zero-dependency CloudWatch log retrieval

**Key Decisions**
- `ZipFile` inline code used for a trivial handler; production functions should use S3 or ECR-backed packages beyond the 4 KB inline limit
- `wait` commands inserted between all sequential CLI steps to prevent non-deterministic pipeline failures

**Outcome**
Stack reached `CREATE_COMPLETE`. Function invoked successfully with `statusCode: 200`. Cold start measured at `87.03 ms` init; warm execution at `1.88 ms`. Cleanup runbook included with log group deletion.

---

### [Lambda Greeting Function Deployment](./lambda-greeting-function-deployment/)

**Quick Summary:** Console-driven Lambda deployment covering IAM role creation, inline code authoring, and synchronous test invocation. Documents the full console workflow with production-relevant configuration notes.

| | |
|---|---|
| **Function Name** | `nautilus-lambda` |
| **Runtime** | Python 3.12 |
| **IAM Role** | `lambda_execution_role` (console-created) |
| **Region** | `us-east-1` |

**Purpose**
Provision a Lambda function through the AWS Management Console, covering the complete workflow from IAM role creation to validated invocation. Relevant for environments without CLI access or IaC tooling.

**Approach**
- IAM role provisioned before function creation; `AWSLambdaBasicExecutionRole` attached as the only managed policy
- Inline code authored directly in the Lambda console editor using `json.dumps()` to serialize the response body, conforming to the API Gateway proxy integration contract even without an API Gateway trigger
- Test event created and invoked synchronously; `statusCode` confirmed as integer `200` (not string `"200"`)

**Key Decisions**
- `json.dumps()` wrapping applied as a production best practice for Lambda response bodies, not a lab requirement
- Region lock enforced manually after each IAM navigation, since IAM resets the region selector to "Global"

**Outcome**
Function invoked with status `Succeeded`. Response body confirmed: `"Welcome to KKE AWS Labs!"`. Console-specific failure modes documented (iframe blocking, region selector drift, Deploy vs. Save distinction).

---

## Technologies and Tools

| Layer | Technology |
|---|---|
| Compute | AWS Lambda (Python 3.12) |
| IaC | AWS CloudFormation (YAML) |
| CLI | AWS CLI v1.40+ |
| Identity | AWS IAM (roles, trust policies, managed policies) |
| Observability | Amazon CloudWatch Logs |
| Packaging | Zip deployment packages, CloudFormation `ZipFile` inline |
| Shell | Bash (heredoc, variable capture, base64 decode) |
| Region | `us-east-1` |

---

## Key Outcomes and Skills Demonstrated

**Serverless Lifecycle Management**
End-to-end Lambda provisioning across three deployment models: CLI, IaC, and console. Each approach is fully documented with validation steps, failure modes, and production-relevant notes.

**Infrastructure as Code**
CloudFormation stack authoring with IAM dependency ordering, named role capabilities, and blocking wait commands for pipeline-safe automation.

**IAM Design**
Consistent application of least-privilege principles across all three projects. Trust policies scoped to `lambda.amazonaws.com` exclusively. No over-permissioned roles.

**Operational Depth**
- `fileb://` vs `file://` distinction for binary uploads
- Zip archive root-level structure to prevent `Runtime.ImportModuleError`
- Dynamic ARN resolution for account-portable scripts
- Cold start vs. execution duration separation in Lambda performance metrics
- Inline log retrieval via `base64 --decode` without CloudWatch Logs console access

**CI/CD Readiness**
The CLI and CloudFormation projects are fully scriptable with no console dependencies. `wait` commands and exit code handling make them safe for automated pipeline integration.

---

## How to Navigate

Each subdirectory is a self-contained project with its own `README.md` covering the full implementation, CLI commands, expected outputs, error handling, and screenshots.

| If you want to... | Go to... |
|---|---|
| See a pipeline-ready CLI deployment | [`lambda-cli-deployment`](./lambda-cli-deployment/) |
| Review IaC-driven provisioning with CloudFormation | [`lambda-cloudformation-deployment-stack`](./lambda-cloudformation-deployment-stack/) |
| Understand the console-based workflow and failure modes | [`lambda-greeting-function-deployment`](./lambda-greeting-function-deployment/) |

---

> Part of the [`cloud-infrastructure-devops-labs`](../../) portfolio. See the root `README.md` for full repository structure and lab index.
