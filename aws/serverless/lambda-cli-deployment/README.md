# AWS Lambda Function Deployment via CLI

[![AWS](https://img.shields.io/badge/AWS-Lambda-FF9900?style=flat-square&logo=amazonaws&logoColor=white)](https://aws.amazon.com/lambda/)
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python&logoColor=white)](https://www.python.org/)
[![Region](https://img.shields.io/badge/Region-us--east--1-232F3E?style=flat-square&logo=amazonaws)](https://aws.amazon.com/)
[![IaC](https://img.shields.io/badge/Deployment-AWS_CLI-232F3E?style=flat-square&logo=amazonaws)](https://aws.amazon.com/cli/)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Implementation](#implementation)
  - [Phase 1: Environment Verification](#phase-1-environment-verification)
  - [Phase 2: Python Handler Authoring](#phase-2-python-handler-authoring)
  - [Phase 3: Deployment Package Creation](#phase-3-deployment-package-creation)
  - [Phase 4: IAM Role Resolution](#phase-4-iam-role-resolution)
  - [Phase 5: Lambda Function Provisioning](#phase-5-lambda-function-provisioning)
  - [Phase 6: Function Validation](#phase-6-function-validation)
- [Resolution and Outcomes](#resolution-and-outcomes)
- [Key Learnings](#key-learnings)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

This document details the end-to-end provisioning of an **AWS Lambda function** using the **AWS CLI** exclusively, without touching the AWS Management Console. The function returns a custom HTTP-style greeting payload with a `200` status code, demonstrating the full serverless deployment lifecycle in a repeatable, scriptable manner.

This pattern forms the foundation for any CI/CD pipeline that needs to programmatically create, update, or manage Lambda functions at scale.

---

## Problem Statement

The Nautilus DevOps team required a serverless function deployed **entirely from the command line** to:

1. Eliminate console-dependent workflows and enable pipeline-ready deployments
2. Validate that the `lambda_execution_role` IAM role is correctly configured for Lambda invocation
3. Demonstrate familiarity with AWS CLI-based Lambda lifecycle management (create, verify, invoke)
4. Return a deterministic response body (`Welcome to KKE AWS Labs!`) with HTTP status `200`

**Constraints:**
- Function name must be `xfusion-lambda-cli`
- Runtime must be `Python`
- IAM Role must be `lambda_execution_role` (pre-existing, not created in this task)
- Deployment package must be a `.zip` archive named `function.zip`
- All operations must target `us-east-1`

---

## Architecture

```
Developer Workstation (aws-client host)
         |
         |  AWS CLI
         v
+---------------------------+
|   IAM Role Resolution     |
|  lambda_execution_role    |
+---------------------------+
         |
         v
+---------------------------+        +---------------------------+
|   Lambda Function         | -----> |   CloudWatch Logs         |
|   xfusion-lambda-cli      |        |  /aws/lambda/xfusion-     |
|   Runtime: python3.12     |        |   lambda-cli              |
|   Handler: lambda_        |        +---------------------------+
|   function.lambda_handler |
|   Memory: 128 MB          |
|   Timeout: 3s             |
+---------------------------+
         |
         v
  Response Payload:
  { "statusCode": 200,
    "body": "Welcome to KKE AWS Labs!" }
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | Installed and configured on `aws-client` host |
| AWS Region | `us-east-1` |
| IAM User | `kk_labs_user_767995` with Lambda and IAM read permissions |
| IAM Role | `lambda_execution_role` must exist prior to deployment |
| Python knowledge | Basic understanding of Lambda handler signature |
| Shell | Bash-compatible (zsh used in lab) |

---

## Implementation

### Phase 1: Environment Verification

**Problem:** Before provisioning any cloud resource, the active AWS identity and configured region must be confirmed to prevent accidental cross-account or cross-region deployments.

**Resolution:** Verify the caller identity and region using `aws sts` and `aws configure`.

```bash
aws sts get-caller-identity
aws configure get region
```

**Expected Output:**

```json
{
    "UserId": "AIDAYLRLBAK6GUI2RLGUH",
    "Account": "574542119612",
    "Arn": "arn:aws:iam::574542119612:user/kk_labs_user_767995"
}
```

```
us-east-1
```

> **Validation gate:** If the region is not `us-east-1` or the account ID does not match the lab environment, stop and run `aws configure set region us-east-1` before proceeding.

***Screenshot: Terminal output showing caller identity and region confirmation***
<img width="1032" height="573" alt="image" src="https://github.com/user-attachments/assets/ca20e457-992b-47a1-8bd4-390f898e741d" />

---

### Phase 2: Python Handler Authoring

**Problem:** Lambda requires a handler file that conforms to a specific signature: `def function_name(event, context)`. The file name and function name together form the handler string used at deployment time.

**Resolution:** Create `lambda_function.py` using a heredoc to avoid editor dependencies on the remote host.

```bash
mkdir -p ~/lambda-cli-deployment && cd ~/lambda-cli-deployment

cat > lambda_function.py << 'EOF'
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Welcome to KKE AWS Labs!'
    }
EOF
```

**Verify file contents:**

```bash
cat lambda_function.py
```

**Expected Output:**

```python
def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'body': 'Welcome to KKE AWS Labs!'
    }
```

> **Key detail:** The single quotes around `'EOF'` in the heredoc prevent shell variable interpolation inside the block, ensuring the Python source is written verbatim.

***Screenshot: Terminal showing cat output confirming lambda_function.py contents***
<img width="1035" height="615" alt="image" src="https://github.com/user-attachments/assets/2f03d71a-860a-44ed-ae9c-ba3d3e535b0a" />

---

### Phase 3: Deployment Package Creation

**Problem:** AWS Lambda does not accept raw `.py` files via the CLI. The source code must be packaged as a `.zip` archive. A critical failure mode is zipping the file from a parent directory, which creates a path prefix inside the archive that causes Lambda to fail with `Runtime.ImportModuleError`.

**Resolution:** Zip the file from within the same directory to ensure `lambda_function.py` sits at the root of the archive.

```bash
zip function.zip lambda_function.py
```

**Verify archive structure:**

```bash
unzip -l function.zip
```

**Expected Output:**

```
Archive:  function.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
      125  2026-03-07 01:21   lambda_function.py
---------                     -------
      125                     1 file
```

> **Critical check:** The `Name` column must show `lambda_function.py` with no directory prefix. Any path prefix (e.g., `lambda-cli-deployment/lambda_function.py`) will cause a silent module import failure at invocation time.

***Screenshot: Terminal showing zip creation and unzip -l verification output***
<img width="1032" height="843" alt="image" src="https://github.com/user-attachments/assets/806c9953-1ec4-4d8f-b27c-e1a75557cf33" />

---

### Phase 4: IAM Role Resolution

**Problem:** The `aws lambda create-function` command requires the full ARN of the execution role, not just the role name. Hardcoding ARNs introduces account-portability issues and human error risk.

**Resolution:** Dynamically resolve the ARN at runtime using `aws iam get-role` and capture it in a shell variable.

```bash
ROLE_ARN=$(aws iam get-role \
  --role-name lambda_execution_role \
  --query 'Role.Arn' \
  --output text)

echo "Role ARN: $ROLE_ARN"
```

**Expected Output:**

```
Role ARN: arn:aws:iam::574542119612:role/lambda_execution_role
```

> **Why this matters:** Using `$ROLE_ARN` instead of a hardcoded ARN makes the script portable across accounts and eliminates a common source of `InvalidParameterValueException` errors during function creation.

***Screenshot: Terminal showing ROLE_ARN variable assignment and echo confirmation***
<img width="1037" height="860" alt="image" src="https://github.com/user-attachments/assets/c44a733f-e0eb-47de-a6d5-4ce2786e1106" />

---

### Phase 5: Lambda Function Provisioning

**Problem:** The function must be created with the exact name, runtime, handler string, IAM role, and deployment package specified by the task requirements. Any mismatch in handler format or zip loading method results in immediate or deferred failures.

**Resolution:** Execute `aws lambda create-function` with all required parameters, using `fileb://` (binary file prefix) for the zip file argument.

```bash
aws lambda create-function \
  --function-name xfusion-lambda-cli \
  --runtime python3.12 \
  --role "$ROLE_ARN" \
  --handler lambda_function.lambda_handler \
  --zip-file fileb://function.zip \
  --region us-east-1
```

**Expected Response (abbreviated):**

```json
{
    "FunctionName": "xfusion-lambda-cli",
    "FunctionArn": "arn:aws:lambda:us-east-1:574542119612:function:xfusion-lambda-cli",
    "Runtime": "python3.12",
    "Role": "arn:aws:iam::574542119612:role/lambda_execution_role",
    "Handler": "lambda_function.lambda_handler",
    "State": "Pending",
    "StateReason": "The function is being created.",
    "PackageType": "Zip",
    "LogGroup": "/aws/lambda/xfusion-lambda-cli"
}
```

**Parameter Reference:**

| Parameter | Value | Rationale |
|---|---|---|
| `--function-name` | `xfusion-lambda-cli` | Required by task specification |
| `--runtime` | `python3.12` | Current stable Python Lambda runtime |
| `--role` | `$ROLE_ARN` | Dynamically resolved ARN from Phase 4 |
| `--handler` | `lambda_function.lambda_handler` | `<filename_without_extension>.<function_name>` |
| `--zip-file` | `fileb://function.zip` | `fileb://` required for binary content; `file://` causes base64 error |
| `--region` | `us-east-1` | Explicit region to prevent misconfiguration |

> **State note:** `"State": "Pending"` is expected immediately after creation. AWS asynchronously initializes the execution environment. The function transitions to `Active` within seconds.

***Screenshots: Full JSON response from aws lambda create-function***
<img width="1037" height="848" alt="image" src="https://github.com/user-attachments/assets/ffca049f-1574-4da3-9213-073f0570a117" />
<img width="1034" height="858" alt="image" src="https://github.com/user-attachments/assets/796a9f5a-af3c-4e25-9c84-abcee72cca4a" />

---

### Phase 6: Function Validation

**Problem:** A successful `create-function` API response confirms the control plane accepted the request, but does not confirm the function is invocable. The state must be `Active` and the invocation response must return the correct payload.

**Resolution:** Poll function state, then invoke and inspect the response payload.

**Step 6a: Confirm Active State**

```bash
aws lambda get-function \
  --function-name xfusion-lambda-cli \
  --region us-east-1 \
  --query 'Configuration.[State,StateReason]' \
  --output text
```

**Expected Output:**

```
Active  None
```

**Step 6b: Invoke and Verify Response**

```bash
aws lambda invoke \
  --function-name xfusion-lambda-cli \
  --region us-east-1 \
  response.json && cat response.json
```

**Expected Output:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
{"statusCode": 200, "body": "Welcome to KKE AWS Labs!"}
```

> **Output structure explained:** The first JSON block (`StatusCode: 200`) is the Lambda service invocation metadata. The second line is the actual function return value written to `response.json`. Both must be present for a successful validation.

***Screenshot: Terminal showing Active state confirmation and invoke response with correct body***

---

## Resolution and Outcomes

All task requirements were met in full:

| Requirement | Result |
|---|---|
| Python script `lambda_function.py` created | Confirmed via `cat` |
| Script zipped as `function.zip` with correct root-level structure | Confirmed via `unzip -l` |
| Lambda function `xfusion-lambda-cli` created | Confirmed via create-function response |
| Runtime `python3.12` used | Confirmed in function metadata |
| IAM role `lambda_execution_role` attached | Confirmed via ARN in response |
| Function state transitioned to `Active` | Confirmed via `get-function` |
| Invocation returned `statusCode: 200` and correct body | Confirmed via `invoke` |

**Function ARN (provisioned):**
```
arn:aws:lambda:us-east-1:574542119612:function:xfusion-lambda-cli
```

**CloudWatch Log Group (auto-created):**
```
/aws/lambda/xfusion-lambda-cli
```

***Screenshot: Final terminal state showing Active function and successful invocation output***

---

## Key Learnings

**1. `fileb://` vs `file://` for zip uploads**
The `fileb://` prefix instructs the AWS CLI to treat the file as raw binary. Using `file://` causes the CLI to base64-encode the content before sending, resulting in an `InvalidParameterValueException`. Always use `fileb://` for `.zip` deployment packages.

**2. Zip structure determines handler resolution**
Lambda resolves the handler string `lambda_function.lambda_handler` by looking for `lambda_function.py` at the root of the extracted zip. Creating the zip from a parent directory embeds a subdirectory path, which silently breaks module import at cold start.

**3. Pending state is not an error**
Lambda's `create-function` is asynchronous at the data plane level. The `Pending` state indicates the control plane accepted the request and is initializing the execution environment. Invoking during `Pending` state returns a `ResourceConflictException`. Always verify `Active` before invoking in automated pipelines.

**4. Dynamic ARN resolution over hardcoding**
Capturing IAM role ARNs via `aws iam get-role --query 'Role.Arn'` makes deployment scripts portable across accounts. This is the standard pattern in production CI/CD pipelines and IaC tooling.

**5. Heredoc with quoted delimiter prevents interpolation**
Using `<< 'EOF'` (single-quoted) instead of `<< EOF` ensures the shell does not interpret `$` characters inside the heredoc block. This is critical when writing Python code containing string literals or dictionary values.

---

## Troubleshooting

| Error | Root Cause | Resolution |
|---|---|---|
| `ResourceNotFoundException` on IAM role | Role name misspelled or does not exist in account | Run `aws iam get-role --role-name lambda_execution_role` to verify existence |
| `InvalidParameterValueException: handler` | Handler string format incorrect | Must follow `<filename_no_extension>.<function_name>` pattern exactly |
| `InvalidParameterValueException: ZipFile` | Used `file://` instead of `fileb://` | Replace `file://` with `fileb://` in the `--zip-file` argument |
| `Runtime.ImportModuleError` at invocation | `lambda_function.py` is nested inside a subdirectory in the zip | Re-zip from within the directory containing the `.py` file |
| `ResourceConflictException` on invoke | Function is still in `Pending` state | Wait for `Active` state; poll with `get-function` before invoking |
| `AccessDeniedException` on create-function | IAM user lacks `lambda:CreateFunction` permission | Confirm the lab user has the required Lambda permissions attached |

---

## References

- [AWS Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [AWS CLI Lambda Reference](https://docs.aws.amazon.com/cli/latest/reference/lambda/)
- [Lambda Deployment Packages](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-package.html)
- [IAM Roles for Lambda](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [Lambda Runtimes](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html)

---

<img width="1035" height="573" alt="image" src="https://github.com/user-attachments/assets/ca8b38a6-54dc-4c7d-bbb6-ce1f6a41288c" />

<img width="1032" height="546" alt="image" src="https://github.com/user-attachments/assets/0307d279-6827-45a7-9f42-9b517fa433da" />
<img width="1036" height="538" alt="image" src="https://github.com/user-attachments/assets/7ba3133c-cfec-4ca2-8312-b77cf51f6273" />

<img width="1030" height="707" alt="image" src="https://github.com/user-attachments/assets/9b40a1c4-5f0c-4998-b5f2-1358495a8732" />



<img width="1033" height="522" alt="image" src="https://github.com/user-attachments/assets/b6b7e7b8-66dd-49f3-852a-c54c96216b20" />
<img width="1035" height="355" alt="image" src="https://github.com/user-attachments/assets/deabb7b5-71a6-4347-a2d2-454d40cab6e0" />

