# Deploying AWS Lambda via CloudFormation IaC Stack

![AWS](https://img.shields.io/badge/AWS-CloudFormation-orange?style=for-the-badge&logo=amazon-aws)
![Lambda](https://img.shields.io/badge/AWS-Lambda-FF9900?style=for-the-badge&logo=aws-lambda)
![IAM](https://img.shields.io/badge/AWS-IAM-red?style=for-the-badge&logo=amazon-aws)
![Python](https://img.shields.io/badge/Python-3.12-blue?style=for-the-badge&logo=python)
![IaC](https://img.shields.io/badge/IaC-CloudFormation-yellow?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Production--Ready-brightgreen?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Solution Walkthrough](#solution-walkthrough)
  - [Step 1: Verify AWS Identity and CLI Access](#step-1-verify-aws-identity-and-cli-access)
  - [Step 2: Author the CloudFormation Template](#step-2-author-the-cloudformation-template)
  - [Step 3: Validate the CloudFormation Template](#step-3-validate-the-cloudformation-template)
  - [Step 4: Deploy the CloudFormation Stack](#step-4-deploy-the-cloudformation-stack)
  - [Step 5: Confirm Stack Creation Status](#step-5-confirm-stack-creation-status)
  - [Step 6: Verify the Lambda Function and IAM Role](#step-6-verify-the-lambda-function-and-iam-role)
  - [Step 7: Invoke the Lambda Function and Inspect Logs](#step-7-invoke-the-lambda-function-and-inspect-logs)
  - [Step 8: Validate the Function Response Payload](#step-8-validate-the-function-response-payload)
- [CloudFormation Template Reference](#cloudformation-template-reference)
- [IAM Role and Permissions Design](#iam-role-and-permissions-design)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Cleanup](#cleanup)
- [Environment Details](#environment-details)

---

## Overview

This repository documents the end-to-end process of provisioning an AWS Lambda function and its associated IAM execution role using AWS CloudFormation as the Infrastructure as Code (IaC) framework. The deployment follows strict GitOps and IaC principles, eliminating all manual console-based resource creation and ensuring full reproducibility, auditability, and drift prevention.

The Lambda function is written in Python 3.12, deployed via a CloudFormation stack named `devops-lambda-app`, and validated by direct invocation through the AWS CLI with log verification.

> **Target Audience:** DevOps Engineers, Cloud Architects, Platform Engineers, and Site Reliability Engineers operating in AWS environments at scale.

---

## Problem Statement

**Context:** The Nautilus DevOps team requires a Lambda function deployed through a fully declarative, version-controlled CloudFormation stack. Manual console deployments are prohibited. All resources must be reproducible, auditable, and least-privilege by design.

**Requirements:**

| Requirement | Value |
|---|---|
| CloudFormation Template Path | `/root/devops-lambda.yml` |
| Stack Name | `devops-lambda-app` |
| Lambda Function Name | `devops-lambda` |
| Runtime | Python 3.12 |
| Function Response Body | `Welcome to KKE AWS Labs!` |
| HTTP Status Code | `200` |
| IAM Execution Role Name | `lambda_execution_role` |
| Target Region | `us-east-1` |

**Problem:** Without IaC, Lambda deployments are error-prone, undocumented, and non-repeatable. IAM roles created manually often accumulate excessive permissions over time. This solution enforces configuration as code from Day 1.

---

## Architecture

```
 AWS Account: 046473746767
 Region: us-east-1
 
 +-----------------------------------------------------+
 |          CloudFormation Stack: devops-lambda-app    |
 |                                                     |
 |   +------------------+     +---------------------+ |
 |   |  IAM Role        |---->|  Lambda Function    | |
 |   |  lambda_         |     |  devops-lambda      | |
 |   |  execution_role  |     |  Runtime: py3.12    | |
 |   |                  |     |  Handler: index.    | |
 |   | ManagedPolicy:   |     |  handler            | |
 |   | AWSLambdaBasic   |     |                     | |
 |   | ExecutionRole    |     |  Returns: 200 OK    | |
 |   +------------------+     +---------------------+ |
 |                                                     |
 +-----------------------------------------------------+
          ^
          |
    AWS CLI / CloudFormation API
          |
    DevOps Engineer (aws-client host)
```

**Resource dependency chain:** `IAM Role` is created first via `!GetAtt` reference. CloudFormation automatically handles dependency ordering. The Lambda function assumes the role only after the role is fully provisioned.

---

## Prerequisites

Before executing this runbook, confirm the following are in place:

| Prerequisite | Minimum Version | Verification Command |
|---|---|---|
| AWS CLI | v1.40+ | `aws --version` |
| Python (botocore dependency) | 3.10+ | `python3 --version` |
| IAM Permissions | CloudFormation full + IAM role create + Lambda full | `aws sts get-caller-identity` |
| Target Region | us-east-1 | Confirm via AWS CLI config |
| AWS Credentials | Valid, non-expired | `aws sts get-caller-identity` |

> **Security Note:** All operations are scoped to the least-privilege user `kk_labs_user_879191` under account `046473746767`. Never use root credentials for programmatic access.

---

## Repository Structure

```
.
+-- devops-lambda.yml        # CloudFormation IaC template
+-- README.md                # This documentation
+-- output.json              # Lambda invocation response payload (generated at runtime)
```

---

## Solution Walkthrough

### Step 1: Verify AWS Identity and CLI Access

**Purpose:** Confirm the CLI is configured correctly, the assumed identity has sufficient permissions, and the correct AWS account is targeted before any resource provisioning begins.

```bash
aws --version
```

**Expected Output:**
```
aws-cli/1.40.19 Python/3.10.17 Linux/6.8.0-90-generic botocore/1.38.20
```

```bash
aws sts get-caller-identity
```

**Expected Output:**
```json
{
    "UserId": "AIDAQVUQNDFHRNEBI3EYO",
    "Account": "046473746767",
    "Arn": "arn:aws:iam::046473746767:user/kk_labs_user_879191"
}
```

> **Checkpoint:** Confirm the `Account` ID and `Arn` match your target environment before proceeding. A mismatch here means your CLI profile is pointed at the wrong account.

**SCREENSHOT:** 

<img width="1033" height="432" alt="image" src="https://github.com/user-attachments/assets/fca053c2-a873-4cb4-b384-6bfccc7290a2" />

>Terminal output showing `aws sts get-caller-identity` with Account ID and ARN clearly visible

---

### Step 2: Author the CloudFormation Template

**Purpose:** Write the declarative IaC template that defines all required resources. This template is the single source of truth for the entire deployment.

Create the template at the specified path using a heredoc to ensure exact content fidelity:

```bash
cat > /root/devops-lambda.yml << 'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudFormation stack to create a Lambda function'

Resources:

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  DevopsLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: devops-lambda
      Runtime: python3.12
      Role: !GetAtt LambdaExecutionRole.Arn
      Handler: index.handler
      Code:
        ZipFile: |
          def handler(event, context):
              print("Welcome to KKE AWS Labs!")
              return {
                  'statusCode': 200,
                  'body': 'Welcome to KKE AWS Labs!'
              }

EOF
```

Verify the file was written correctly:

```bash
cat /root/devops-lambda.yml
```

> **Pro Tip:** Always use a heredoc with `'EOF'` (single-quoted) when writing YAML via the CLI. This prevents shell interpolation of `$` characters, `!` exclamation marks, and backticks that are common in CloudFormation templates.

**SCREENSHOTS:** 

<img width="1030" height="865" alt="image" src="https://github.com/user-attachments/assets/d98c3789-567a-4f5f-97f3-426e8b189d60" />
<img width="1031" height="854" alt="image" src="https://github.com/user-attachments/assets/b58d6ec7-3cbc-4237-aa9d-4ede1e18fe4c" />

>Terminal showing the `cat /root/devops-lambda.yml` output with the full template content rendered correctly]

---

### Step 3: Validate the CloudFormation Template

**Purpose:** Use the CloudFormation API to perform server-side linting and structural validation before deployment. This catches YAML syntax errors, invalid resource types, and missing capability declarations.

```bash
aws cloudformation validate-template \
  --template-body file:///root/devops-lambda.yml \
  --region us-east-1
```

**Expected Output:**
```json
{
    "Parameters": [],
    "Description": "CloudFormation stack to create a Lambda function",
    "Capabilities": [
        "CAPABILITY_NAMED_IAM"
    ],
    "CapabilitiesReason": "The following resource(s) require capabilities: [AWS::IAM::Role]"
}
```

> **Critical:** The `CAPABILITY_NAMED_IAM` flag is explicitly required because the template creates an IAM role with a custom name (`lambda_execution_role`). AWS enforces this acknowledgement to prevent accidental privilege escalation. This flag must be passed during stack creation.

**[SCREENSHOT PLACEHOLDER: Terminal showing the `validate-template` command with the `Capabilities` array output confirming `CAPABILITY_NAMED_IAM`]**

---

### Step 4: Deploy the CloudFormation Stack

**Purpose:** Submit the template to CloudFormation for asynchronous stack creation. The `--capabilities CAPABILITY_NAMED_IAM` flag explicitly acknowledges IAM resource creation.

```bash
aws cloudformation create-stack \
  --stack-name devops-lambda-app \
  --template-body file:///root/devops-lambda.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**Expected Output:**
```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:046473746767:stack/devops-lambda-app/ca76b1f0-2658-11f1-b120-125b2a6475cd"
}
```

Wait for the stack to reach a terminal state before proceeding:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name devops-lambda-app \
  --region us-east-1
```

> **Note:** `aws cloudformation wait stack-create-complete` polls the stack status every 30 seconds (up to 120 times) and exits 0 only on `CREATE_COMPLETE`. Any rollback or failure state causes a non-zero exit code, making it safe to use in CI/CD pipelines.

**[SCREENSHOT PLACEHOLDER: Terminal showing the `create-stack` command with the returned `StackId` ARN]**

---

### Step 5: Confirm Stack Creation Status

**Purpose:** Programmatically verify the stack reached `CREATE_COMPLETE` status. This is the authoritative confirmation that all resources were provisioned successfully.

```bash
aws cloudformation describe-stacks \
  --stack-name devops-lambda-app \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus'
```

**Expected Output:**
```
"CREATE_COMPLETE"
```

> **Incident Prevention:** Never skip this verification step in automated pipelines. Stacks can reach `CREATE_COMPLETE` at the stack level while individual resources are in a degraded state in edge cases. Follow this with resource-level verification (Steps 6+).

**[SCREENSHOT PLACEHOLDER: Terminal showing `describe-stacks` query returning `"CREATE_COMPLETE"` status]**

---

### Step 6: Verify the Lambda Function and IAM Role

**Purpose:** Confirm the Lambda function was provisioned with the exact runtime, handler, and IAM role specified in the template. This is a resource-level audit step.

**Verify Lambda function configuration:**

```bash
aws lambda get-function \
  --function-name devops-lambda \
  --region us-east-1 \
  --query 'Configuration.[FunctionName,Runtime,Role]'
```

**Expected Output:**
```json
[
    "devops-lambda",
    "python3.12",
    "arn:aws:iam::046473746767:role/lambda_execution_role"
]
```

**Verify the IAM execution role exists:**

```bash
aws iam get-role \
  --role-name lambda_execution_role \
  --query 'Role.RoleName'
```

**Expected Output:**
```
"lambda_execution_role"
```

> **Security Validation:** Confirm the role ARN in the Lambda configuration matches the expected pattern `arn:aws:iam::<account_id>:role/lambda_execution_role`. Any deviation indicates configuration drift or an unintended role substitution.

**[SCREENSHOT PLACEHOLDER: Terminal showing `get-function` output with FunctionName, Runtime `python3.12`, and the IAM Role ARN all confirmed]**

**[SCREENSHOT PLACEHOLDER: Terminal showing `get-role` output returning `"lambda_execution_role"`]**

---

### Step 7: Invoke the Lambda Function and Inspect Logs

**Purpose:** Execute a live invocation of the Lambda function and capture the CloudWatch log output inline using the `--log-type Tail` parameter. This verifies runtime behavior, not just provisioning.

```bash
aws lambda invoke \
  --function-name devops-lambda \
  --region us-east-1 \
  --log-type Tail \
  output.json \
  --query 'LogResult' \
  --output text | base64 --decode
```

**Expected Log Output:**
```
START RequestId: 3d3edf59-17a7-4491-b493-47533c6489b7 Version: $LATEST
Welcome to KKE AWS Labs!
END RequestId: 3d3edf59-17a7-4491-b493-47533c6489b7
REPORT RequestId: 3d3edf59-17a7-4491-b493-47533c6489b7  Duration: 1.88 ms  Billed Duration: 89 ms  Memory Size: 128 MB  Max Memory Used: 36 MB  Init Duration: 87.03 ms
```

> **Performance Observation:** The `Init Duration: 87.03 ms` is the cold start overhead for Python 3.12 with a 128 MB memory allocation. Subsequent invocations (warm starts) will run at the `Duration: 1.88 ms` baseline. For latency-critical workloads, consider Lambda Provisioned Concurrency.

> **Log Decoding:** `--log-type Tail` returns the last 4 KB of execution logs as a base64-encoded string in the API response. Piping through `base64 --decode` renders it human-readable. This is especially useful in environments without direct CloudWatch Logs access.

**[SCREENSHOT PLACEHOLDER: Terminal showing the full decoded log output including `START`, the `Welcome to KKE AWS Labs!` print statement, `END`, and the `REPORT` line with duration metrics]**

---

### Step 8: Validate the Function Response Payload

**Purpose:** Inspect the synchronous response payload written to `output.json` to confirm the Lambda function returned the correct `statusCode` and `body`.

```bash
cat output.json
```

**Expected Output:**
```json
{"statusCode": 200, "body": "Welcome to KKE AWS Labs!"}
```

> **Contract Verification:** The response payload is the Lambda function contract. `statusCode: 200` and the exact body string `Welcome to KKE AWS Labs!` must match requirements verbatim. Any deviation (typo, case mismatch, extra whitespace) is a functional defect.

**[SCREENSHOT PLACEHOLDER: Terminal showing `cat output.json` with the exact JSON payload `{"statusCode": 200, "body": "Welcome to KKE AWS Labs!"}`]**

---

## CloudFormation Template Reference

### Full Template: `/root/devops-lambda.yml`

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudFormation stack to create a Lambda function'

Resources:

  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  DevopsLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: devops-lambda
      Runtime: python3.12
      Role: !GetAtt LambdaExecutionRole.Arn
      Handler: index.handler
      Code:
        ZipFile: |
          def handler(event, context):
              print("Welcome to KKE AWS Labs!")
              return {
                  'statusCode': 200,
                  'body': 'Welcome to KKE AWS Labs!'
              }
```

### Resource Dependency Map

| Resource Logical ID | Type | Depends On | Key Properties |
|---|---|---|---|
| `LambdaExecutionRole` | `AWS::IAM::Role` | None | `RoleName: lambda_execution_role` |
| `DevopsLambda` | `AWS::Lambda::Function` | `LambdaExecutionRole` | `Runtime: python3.12`, `Handler: index.handler` |

---

## IAM Role and Permissions Design

### Principle of Least Privilege

The `lambda_execution_role` is granted only the AWS managed policy `AWSLambdaBasicExecutionRole`, which provides the minimum permissions required for Lambda execution:

| Permission | Purpose |
|---|---|
| `logs:CreateLogGroup` | Create a CloudWatch log group for the function |
| `logs:CreateLogStream` | Create log streams within the group |
| `logs:PutLogEvents` | Write log events (function output) to CloudWatch |

### Trust Policy

The role's trust policy restricts assumption to the `lambda.amazonaws.com` service principal exclusively. No IAM users, other services, or cross-account entities can assume this role.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

> **Security Design Note:** This trust policy enforces service-scoped role assumption. Expanding it to include `sts:AssumeRole` for user principals or wildcard services is a security misconfiguration and should be flagged in any IAM policy review.

---

## Best Practices

### IaC and CloudFormation

* **Always validate before deploy.** Use `aws cloudformation validate-template` as a mandatory gate before every `create-stack` or `update-stack`. Treat a failed validation as a hard blocker, not a warning.
* **Use `wait` commands in pipelines.** Never proceed to resource verification without confirming `CREATE_COMPLETE`. Race conditions in automation cause false positives that surface as production incidents.
* **Use `CAPABILITY_NAMED_IAM` explicitly.** Never use `CAPABILITY_IAM` when creating named roles. The named variant forces explicit intent and prevents accidental namespace collisions.
* **Separate IAM resources into their own stack when possible.** For production, IAM roles and policies are better managed in a dedicated IAM stack with restricted update permissions, referenced via `Fn::ImportValue`.
* **Use `DeletionPolicy: Retain` on stateful resources.** Even for Lambda functions, consider retaining associated CloudWatch Log Groups to preserve audit trails on stack deletion.
* **Parameterize templates for reuse.** Replace hardcoded values (`FunctionName`, `RoleName`, `Runtime`) with `Parameters` and `Mappings` blocks to enable environment-specific deployments without template duplication.

### Lambda Function Design

* **Use `ZipFile` only for trivial functions.** Inline code via `ZipFile` is convenient for demos and bootstrapping but has a 4 KB limit. Production functions must use S3 or ECR-based deployment packages.
* **Pin the runtime version explicitly.** `python3.12` is specified by exact version. Never use generic aliases. AWS deprecates runtimes on a fixed schedule and will force-disable them without notice.
* **Always set `MemorySize` and `Timeout` explicitly.** The defaults (128 MB, 3 seconds) are rarely correct for production workloads. Omitting them creates invisible configuration debt.
* **Emit structured logs.** Replace `print()` statements with Python's `logging` module using JSON-formatted output for CloudWatch Logs Insights compatibility.
* **Tag all Lambda functions.** Add a `Tags` block to the Lambda resource for cost allocation, ownership, and environment tracking.

### Security

* **Never embed credentials in CloudFormation templates.** Use AWS Secrets Manager references (`{{resolve:secretsmanager:...}}`) or SSM Parameter Store for sensitive values.
* **Scope IAM roles to the minimum required managed policies.** `AWSLambdaBasicExecutionRole` is correct here. Avoid `AdministratorAccess` or broad service policies.
* **Enable AWS CloudTrail.** All `aws cloudformation` and `aws lambda` API calls are audited by CloudTrail. Ensure it is enabled in the target account and region.
* **Apply resource-based policies to Lambda where appropriate.** For functions invoked by other AWS services (API Gateway, S3, SNS), add a Lambda resource policy rather than relying solely on execution role permissions.

### Operational Excellence

* **Version control the CloudFormation template.** Commit `devops-lambda.yml` to a Git repository immediately after creation. Every change must be a reviewed pull request.
* **Use `--log-type Tail` for synchronous invocation debugging.** This retrieves the last 4 KB of logs inline without requiring CloudWatch Logs access, accelerating root cause analysis.
* **Decode base64 logs in the same pipeline step.** Piping `--output text | base64 --decode` inline eliminates a manual decode step and keeps the debug loop tight.
* **Store invocation payloads for audit.** `output.json` created by `aws lambda invoke` is the authoritative record of function output. Archive it for compliance-sensitive workloads.

---

## Lessons Learned

### 1. `CAPABILITY_NAMED_IAM` Is Non-Negotiable for Named Roles

**Issue:** Omitting `--capabilities CAPABILITY_NAMED_IAM` during `create-stack` causes an immediate `InsufficientCapabilitiesException` failure. This is a common stumbling block for engineers new to CloudFormation IAM deployments.

**Resolution:** Always include `CAPABILITY_NAMED_IAM` when any `AWS::IAM::Role`, `AWS::IAM::User`, or `AWS::IAM::Group` resource uses an explicit `RoleName`, `UserName`, or `GroupName` property. If the template validation output shows `CAPABILITY_NAMED_IAM` in the `Capabilities` array, the deploy command must include it.

### 2. Template Validation Is Not Full Semantic Validation

**Issue:** `aws cloudformation validate-template` only verifies template syntax and structure. It does not validate IAM policy ARNs, Lambda runtime availability, or handler format. A template can pass validation and still fail during stack creation.

**Resolution:** Treat validation as a syntax gate, not a deployment guarantee. Always follow validation with stack creation and confirm `CREATE_COMPLETE` via `describe-stacks`.

### 3. `ZipFile` Indentation Is YAML-Sensitive

**Issue:** The Python code block under `ZipFile: |` is a YAML literal block scalar. Incorrect indentation of the Python code (too many or too few leading spaces) causes a malformed deployment package and results in a `Runtime.ImportModuleError` on invocation.

**Resolution:** Ensure the Python handler code is indented exactly 10 spaces relative to the root key (6 spaces for template depth, 4 for Python indentation). Validate by running `cat` on the template and visually confirming alignment before deployment.

### 4. `base64 --decode` Is Required for Inline Log Retrieval

**Issue:** The `LogResult` field returned by `aws lambda invoke --log-type Tail` is a base64-encoded string. Without decoding, the log is unreadable and engineers may incorrectly conclude logging is broken.

**Resolution:** Always pipe the `--query 'LogResult' --output text` result through `| base64 --decode` to render human-readable CloudWatch log output inline in the terminal.

### 5. Cold Start Duration Is Separable from Execution Duration

**Issue:** Engineers frequently misread the `REPORT` line and treat `Billed Duration: 89 ms` as the function's execution time. The `Init Duration: 87.03 ms` is the Lambda cold start overhead, which is billed on the first invocation of each execution environment.

**Resolution:** Evaluate function performance using `Duration` (actual handler execution time, here 1.88 ms) separately from `Init Duration` (cold start, here 87.03 ms). Cold starts are eliminated on subsequent warm invocations within the same execution environment lifetime.

### 6. `wait stack-create-complete` Blocks Automation Race Conditions

**Issue:** Scripted pipelines that query stack status immediately after `create-stack` may read intermediate states (`CREATE_IN_PROGRESS`) and proceed with downstream steps against resources that are not yet available, causing brittle and non-deterministic failures.

**Resolution:** Always insert `aws cloudformation wait stack-create-complete` between `create-stack` and any downstream resource verification steps. This command is a blocking poll that returns only when the stack reaches a terminal state, making it safe for scripted and CI/CD automation.

---

## Cleanup

To avoid incurring ongoing charges, delete the CloudFormation stack and all associated resources:

```bash
aws cloudformation delete-stack \
  --stack-name devops-lambda-app \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name devops-lambda-app \
  --region us-east-1

echo "Stack deletion complete. All resources removed."
```

> **Warning:** Stack deletion will remove the Lambda function, IAM role, and all associated resources. CloudWatch Log Groups created by the Lambda function are NOT managed by the stack and must be deleted separately if required.

```bash
aws logs delete-log-group \
  --log-group-name /aws/lambda/devops-lambda \
  --region us-east-1
```

---

## Environment Details

| Property | Value |
|---|---|
| AWS Account ID | `046473746767` |
| IAM User | `kk_labs_user_879191` |
| Region | `us-east-1` |
| AWS CLI Version | `1.40.19` |
| Python Version | `3.10.17` |
| botocore Version | `1.38.20` |
| OS | `Linux/6.8.0-90-generic` |
| Lambda Runtime | `python3.12` |
| Stack Name | `devops-lambda-app` |
| Function Name | `devops-lambda` |
| IAM Role Name | `lambda_execution_role` |
| Template Path | `/root/devops-lambda.yml` |

---






<img width="1030" height="864" alt="image" src="https://github.com/user-attachments/assets/a52f195a-f479-46fc-9e8b-9cfe463ae52c" />
<img width="1035" height="805" alt="image" src="https://github.com/user-attachments/assets/7d200a72-c3eb-4e7b-b22f-2595a78ed93d" />
