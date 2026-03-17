# DynamoDB Task Management System

## Provisioning a Serverless NoSQL Table with AWS CLI on Amazon DynamoDB

![AWS DynamoDB](https://img.shields.io/badge/AWS-DynamoDB-FF9900?style=for-the-badge&logo=amazondynamodb&logoColor=white)
![AWS CLI](https://img.shields.io/badge/AWS_CLI-v1.40.19-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-28a745?style=for-the-badge)
![Region](https://img.shields.io/badge/Region-us--east--1-0052CC?style=for-the-badge)

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Verification](#environment-verification)
- [Phase 1: AWS CLI and Credentials Validation](#phase-1-aws-cli-and-credentials-validation)
- [Phase 2: DynamoDB Table Provisioning](#phase-2-dynamodb-table-provisioning)
- [Phase 3: Data Insertion](#phase-3-data-insertion)
- [Phase 4: Data Verification](#phase-4-data-verification)
- [Validation Summary](#validation-summary)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting Reference](#troubleshooting-reference)

---

## Project Overview

The **Nautilus DevOps Team** required a lightweight, scalable, serverless task tracking solution to manage datacenter operations tasks efficiently. This project documents the end-to-end provisioning of an Amazon DynamoDB table named `datacenter-tasks` using the AWS CLI, including schema definition, data ingestion, and integrity verification, all executed without a management console.

| Property | Value |
|---|---|
| **Table Name** | `datacenter-tasks` |
| **Primary Key** | `taskId` (String) |
| **Billing Mode** | `PAY_PER_REQUEST` (On-Demand) |
| **Region** | `us-east-1` |
| **CLI Version** | `aws-cli/1.40.19` |
| **Runtime** | `Python/3.10.17` |
| **OS** | `Linux/6.8.0-90-generic` |

---

## Problem Statement

### Business Context

Datacenter operations teams frequently manage tasks across multiple engineers and systems with no centralized, low-latency tracking mechanism. Spreadsheets and ticketing systems introduce latency, schema rigidity, and operational overhead that slow DevOps workflows.

### Technical Requirements

1. Create a DynamoDB table named `datacenter-tasks` with `taskId` (string) as the partition key.
2. Insert two operational tasks with descriptive metadata and status fields.
3. Verify both records are persisted with accurate status values (`completed` and `in-progress`).

### Resolution

Provisioned a fully serverless, schema-flexible DynamoDB table using AWS CLI with on-demand billing, eliminating capacity planning overhead. Tasks are stored as flexible JSON items and verified via targeted key lookups.

---

## Architecture

```
Developer Workstation (Linux)
         |
         | AWS CLI (aws-cli/1.40.19)
         |
    [AWS Cloud: us-east-1]
         |
    +----+--------------------+
    |  Amazon DynamoDB        |
    |  Table: datacenter-tasks|
    |  PK: taskId (String)    |
    |  Billing: PAY_PER_REQUEST|
    |                         |
    |  Item 1: taskId = "1"   |
    |  Item 2: taskId = "2"   |
    +-------------------------+
```

---

## Prerequisites

Before beginning, confirm the following are in place:

| Requirement | Minimum Version | Purpose |
|---|---|---|
| AWS CLI | v1.x or v2.x | Primary interface for all AWS operations |
| Python | 3.x | Required runtime for AWS CLI v1 |
| AWS IAM Credentials | N/A | Access Key + Secret Key with DynamoDB permissions |
| AWS Region | `us-east-1` | Target deployment region |
| IAM Permissions Required | `dynamodb:CreateTable`, `dynamodb:PutItem`, `dynamodb:GetItem`, `dynamodb:DescribeTable` | Least-privilege policy |

---

## Environment Verification

All commands were executed in a Linux shell environment with the AWS Cloud indicator visible in the prompt (`on (us-east-1)`), confirming the active cloud context before any provisioning began.

---

## Phase 1: AWS CLI and Credentials Validation

### Step 1.1: Verify AWS CLI Installation

**Objective:** Confirm AWS CLI is installed and identify the exact version to ensure compatibility.

```bash
aws --version
```

**Actual Output:**

```
aws-cli/1.40.19 Python/3.10.17 Linux/6.8.0-90-generic botocore/1.38.20
```

**Result:** AWS CLI v1.40.19 confirmed. Python 3.10.17 runtime confirmed. Botocore 1.38.20 confirmed.

> **Screenshot Placeholder**
> ![AWS CLI Version Check](./screenshots/01-aws-version.png)
> *Caption: Terminal output confirming aws-cli/1.40.19 on Python/3.10.17 running on Linux/6.8.0-90-generic*

---

### Step 1.2: Validate Credentials and Active Region

**Objective:** Confirm that IAM credentials are loaded from the shared credentials file and the active region is `us-east-1`.

```bash
aws configure list
```

**Actual Output:**

```
      Name                    Value             Type    Location
      ----                    -----             ----    --------
   profile                <not set>             None    None
access_key     ****************3CMD shared-credentials-file
secret_key     ****************eaLu shared-credentials-file
    region                us-east-1      config-file    ~/.aws/config
```

**Result:** Credentials sourced from `~/.aws/credentials`. Region `us-east-1` confirmed via `~/.aws/config`. No profile override active.

> **Screenshot Placeholder**
> ![AWS Configure List](./screenshots/02-aws-configure-list.png)
> *Caption: Credentials validated from shared-credentials-file with region set to us-east-1*

---

### Step 1.3: Confirm DynamoDB Connectivity

**Objective:** Validate network access to DynamoDB in `us-east-1` and confirm no pre-existing tables conflict with the planned table name.

```bash
aws dynamodb list-tables --region us-east-1
```

**Actual Output:**

```json
{
    "TableNames": []
}
```

**Result:** DynamoDB endpoint reachable. No existing tables present. Environment is clean for provisioning.

> **Screenshot Placeholder**
> ![DynamoDB List Tables Empty](./screenshots/03-list-tables-empty.png)
> *Caption: Empty TableNames array confirms a clean DynamoDB environment with no pre-existing tables*

---

## Phase 2: DynamoDB Table Provisioning

### Step 2.1: Create the `datacenter-tasks` Table

**Objective:** Create a DynamoDB table with `taskId` as the string partition key and on-demand billing mode to eliminate capacity planning.

```bash
aws dynamodb create-table \
  --table-name datacenter-tasks \
  --attribute-definitions AttributeName=taskId,AttributeType=S \
  --key-schema AttributeName=taskId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Flag Breakdown:**

| Flag | Value | Purpose |
|---|---|---|
| `--table-name` | `datacenter-tasks` | Unique table name within the account and region |
| `--attribute-definitions` | `taskId` / `AttributeType=S` | Declares `taskId` as a String type attribute |
| `--key-schema` | `KeyType=HASH` | Sets `taskId` as the partition key (no sort key required) |
| `--billing-mode` | `PAY_PER_REQUEST` | On-demand billing; scales automatically with zero capacity management |
| `--region` | `us-east-1` | Explicit region targeting to prevent accidental provisioning in wrong region |

**Actual Output:**

```json
{
    "TableDescription": {
        "AttributeDefinitions": [
            {
                "AttributeName": "taskId",
                "AttributeType": "S"
            }
        ],
        "TableName": "datacenter-tasks",
        "KeySchema": [
            {
                "AttributeName": "taskId",
                "KeyType": "HASH"
            }
        ],
        "TableStatus": "CREATING",
        "CreationDateTime": 1773708795.618,
        "ProvisionedThroughput": {
            "NumberOfDecreasesToday": 0,
            "ReadCapacityUnits": 0,
            "WriteCapacityUnits": 0
        },
        "TableSizeBytes": 0,
        "ItemCount": 0,
        "TableArn": "arn:aws:dynamodb:us-east-1:881517795257:table/datacenter-tasks",
        "TableId": "1f4eda79-e98d-43c3-9122-d5cdd1502fda",
        "BillingModeSummary": {
            "BillingMode": "PAY_PER_REQUEST"
        },
        "DeletionProtectionEnabled": false
    }
}
```

**Result:** Table creation initiated. `TableStatus: CREATING`. `TableArn` assigned. `BillingMode: PAY_PER_REQUEST` confirmed. `ReadCapacityUnits` and `WriteCapacityUnits` are `0` as expected for on-demand mode.

> **Screenshot Placeholder**
> ![DynamoDB Table Creating](./screenshots/04-create-table-creating.png)
> *Caption: Full JSON response showing TableStatus CREATING with the assigned TableArn and PAY_PER_REQUEST billing*

---

### Step 2.2: Wait for Table to Reach ACTIVE State

**Objective:** Block execution and poll DynamoDB until the table transitions from `CREATING` to `ACTIVE`. This is a critical gate to prevent `ResourceNotFoundException` errors on subsequent write operations.

```bash
aws dynamodb wait table-exists \
  --table-name datacenter-tasks \
  --region us-east-1
```

**Actual Output:**

```
(no output)
```

**Result:** The command returned silently with no output, which is the expected success behavior. The prompt returned immediately after the table became `ACTIVE`. AWS CLI `wait` commands return exit code `0` on success and exit code `255` on timeout.

> **Screenshot Placeholder**
> ![DynamoDB Wait Table Exists](./screenshots/05-wait-table-exists.png)
> *Caption: Silent return of the wait command confirming table reached ACTIVE state*

---

### Step 2.3: Confirm Table is ACTIVE

**Objective:** Explicitly confirm the `TableStatus` is `ACTIVE` before proceeding to data insertion. Using `--query` extracts only the status field for a clean, unambiguous confirmation.

```bash
aws dynamodb describe-table \
  --table-name datacenter-tasks \
  --region us-east-1 \
  --query "Table.TableStatus"
```

**Actual Output:**

```
"ACTIVE"
```

**Result:** Table status confirmed as `ACTIVE`. Safe to proceed with data insertion.

> **Screenshot Placeholder**
> ![DynamoDB Table ACTIVE](./screenshots/06-describe-table-active.png)
> *Caption: Single-field query output showing "ACTIVE" confirming the table is fully ready for read and write operations*

---

## Phase 3: Data Insertion

### Step 3.1: Insert Task 1 (Completed)

**Objective:** Insert the first task record with `taskId` of `"1"`, a description of `"Learn DynamoDB"`, and a status of `"completed"`.

```bash
aws dynamodb put-item \
  --table-name datacenter-tasks \
  --item '{"taskId": {"S": "1"}, "description": {"S": "Learn DynamoDB"}, "status": {"S": "completed"}}' \
  --region us-east-1
```

**Item Schema:**

| Attribute | DynamoDB Type | Value |
|---|---|---|
| `taskId` | `S` (String) | `"1"` |
| `description` | `S` (String) | `"Learn DynamoDB"` |
| `status` | `S` (String) | `"completed"` |

**Actual Output:**

```
(no output)
```

**Result:** No output is the expected success response for `put-item`. AWS CLI returns nothing on a successful write. A non-zero exit code or an error message would indicate failure.

> **Screenshot Placeholder**
> ![Put Item Task 1](./screenshots/07-put-item-task1.png)
> *Caption: Silent successful insertion of Task 1 with taskId "1" and status "completed"*

---

### Step 3.2: Insert Task 2 (In Progress)

**Objective:** Insert the second task record with `taskId` of `"2"`, a description of `"Build To-Do App"`, and a status of `"in-progress"`.

```bash
aws dynamodb put-item \
  --table-name datacenter-tasks \
  --item '{"taskId": {"S": "2"}, "description": {"S": "Build To-Do App"}, "status": {"S": "in-progress"}}' \
  --region us-east-1
```

**Item Schema:**

| Attribute | DynamoDB Type | Value |
|---|---|---|
| `taskId` | `S` (String) | `"2"` |
| `description` | `S` (String) | `"Build To-Do App"` |
| `status` | `S` (String) | `"in-progress"` |

**Actual Output:**

```
(no output)
```

**Result:** Silent success. Task 2 persisted to DynamoDB.

> **Screenshot Placeholder**
> ![Put Item Task 2](./screenshots/08-put-item-task2.png)
> *Caption: Silent successful insertion of Task 2 with taskId "2" and status "in-progress"*

---

## Phase 4: Data Verification

### Step 4.1: Verify Task 1 Status is `completed`

**Objective:** Retrieve Task 1 by primary key to confirm all attributes are persisted correctly, specifically validating that `status` equals `"completed"`.

```bash
aws dynamodb get-item \
  --table-name datacenter-tasks \
  --key '{"taskId": {"S": "1"}}' \
  --region us-east-1
```

**Actual Output:**

```json
{
    "Item": {
        "description": {
            "S": "Learn DynamoDB"
        },
        "taskId": {
            "S": "1"
        },
        "status": {
            "S": "completed"
        }
    }
}
```

**Result:** All three attributes returned and validated:
- `taskId`: `"1"` confirmed
- `description`: `"Learn DynamoDB"` confirmed
- `status`: `"completed"` confirmed

> **Screenshot Placeholder**
> ![Get Item Task 1 Verified](./screenshots/09-get-item-task1-verified.png)
> *Caption: Full item returned for taskId "1" with status "completed" confirmed*

---

### Step 4.2: Verify Task 2 Status is `in-progress`

**Objective:** Retrieve Task 2 by primary key to confirm all attributes are persisted correctly, specifically validating that `status` equals `"in-progress"`.

```bash
aws dynamodb get-item \
  --table-name datacenter-tasks \
  --key '{"taskId": {"S": "2"}}' \
  --region us-east-1
```

**Actual Output:**

```json
{
    "Item": {
        "description": {
            "S": "Build To-Do App"
        },
        "taskId": {
            "S": "2"
        },
        "status": {
            "S": "in-progress"
        }
    }
}
```

**Result:** All three attributes returned and validated:
- `taskId`: `"2"` confirmed
- `description`: `"Build To-Do App"` confirmed
- `status`: `"in-progress"` confirmed

> **Screenshot Placeholder**
> ![Get Item Task 2 Verified](./screenshots/10-get-item-task2-verified.png)
> *Caption: Full item returned for taskId "2" with status "in-progress" confirmed*

---

## Validation Summary

| Phase | Step | Command | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|
| 1 | CLI Version Check | `aws --version` | CLI version string | `aws-cli/1.40.19` | PASS |
| 1 | Credentials Check | `aws configure list` | Keys + `us-east-1` | Keys from credentials file, region `us-east-1` | PASS |
| 1 | Connectivity Check | `aws dynamodb list-tables` | `{"TableNames": []}` | `{"TableNames": []}` | PASS |
| 2 | Create Table | `aws dynamodb create-table` | `TableStatus: CREATING` | `TableStatus: CREATING` | PASS |
| 2 | Wait for ACTIVE | `aws dynamodb wait table-exists` | Silent return | Silent return | PASS |
| 2 | Confirm ACTIVE | `aws dynamodb describe-table` | `"ACTIVE"` | `"ACTIVE"` | PASS |
| 3 | Insert Task 1 | `aws dynamodb put-item` | No output | No output | PASS |
| 3 | Insert Task 2 | `aws dynamodb put-item` | No output | No output | PASS |
| 4 | Verify Task 1 | `aws dynamodb get-item` | `status: completed` | `status: completed` | PASS |
| 4 | Verify Task 2 | `aws dynamodb get-item` | `status: in-progress` | `status: in-progress` | PASS |

**Overall Result: 10/10 Steps Passed**

---

## Best Practices

### 1. Always Specify `--region` Explicitly

Even when a default region is set in `~/.aws/config`, always pass `--region` in every command. This prevents accidental provisioning in the wrong region, especially in multi-region environments or when switching contexts.

```bash
# Correct approach
aws dynamodb create-table --region us-east-1 ...

# Risky approach (relies on config default)
aws dynamodb create-table ...
```

### 2. Gate Operations with `wait` Commands

Never assume a table is ready immediately after `create-table` returns. DynamoDB tables go through a `CREATING` state that typically lasts a few seconds. Writing to a table in `CREATING` state raises `ResourceNotFoundException`.

```bash
# Always wait before writing
aws dynamodb wait table-exists --table-name datacenter-tasks --region us-east-1
```

### 3. Use `--query` for Targeted Verification

Instead of parsing full JSON responses, use JMESPath `--query` expressions to extract exactly the field you need to verify. This removes ambiguity and is more suitable for scripting and CI/CD pipelines.

```bash
aws dynamodb describe-table \
  --table-name datacenter-tasks \
  --region us-east-1 \
  --query "Table.TableStatus"
```

### 4. Use `PAY_PER_REQUEST` for Variable Workloads

On-demand billing (`PAY_PER_REQUEST`) is the correct default for new tables with unpredictable or low traffic. It eliminates the risk of under-provisioned read/write capacity units and avoids `ProvisionedThroughputExceededException` errors.

### 5. Use DynamoDB Type Descriptors in All CLI Item Operations

DynamoDB CLI requires explicit type descriptors (`{"S": "value"}` for strings, `{"N": "123"}` for numbers). Never pass raw values. Failing to include type descriptors causes `ValidationException` errors.

```json
// Correct
{"taskId": {"S": "1"}, "status": {"S": "completed"}}

// Incorrect (will fail)
{"taskId": "1", "status": "completed"}
```

### 6. Validate Connectivity Before Provisioning

Always run `aws dynamodb list-tables` before any provisioning operation. This validates credentials, network access, and the target region in a single, read-only, non-destructive command.

### 7. Apply Least-Privilege IAM Policies

Scope IAM permissions to only what is required for the task. For this workflow, the minimum required permissions are:

```json
{
  "Effect": "Allow",
  "Action": [
    "dynamodb:CreateTable",
    "dynamodb:DescribeTable",
    "dynamodb:PutItem",
    "dynamodb:GetItem"
  ],
  "Resource": "arn:aws:dynamodb:us-east-1:*:table/datacenter-tasks"
}
```

### 8. Enable Deletion Protection in Production

The table was created with `DeletionProtectionEnabled: false`. For production workloads, always enable deletion protection to prevent accidental table drops.

```bash
aws dynamodb update-table \
  --table-name datacenter-tasks \
  --deletion-protection-enabled \
  --region us-east-1
```

---

## Lessons Learned

### 1. Silent Success is Valid Success in AWS CLI

The `put-item` and `wait` commands return no output on success. This initially appears ambiguous but is the correct AWS CLI behavior. An error would always produce an explicit error message. If a command returns silently with exit code `0`, the operation succeeded.

### 2. `CREATING` Status Requires Explicit Gating

On the first attempt, attempting to write immediately after `create-table` returns would fail. The `wait table-exists` command is essential and should be treated as a mandatory step, not an optional one.

### 3. Explicit Region Targeting Prevents Cross-Region Mistakes

In shared team environments where multiple engineers operate in different regions, relying on config-file defaults creates risk. Hardcoding `--region` in every command is a lightweight safeguard that costs nothing but prevents expensive debugging sessions.

### 4. Type Descriptors are Not Optional in DynamoDB CLI

Unlike higher-level SDKs such as boto3 with the resource interface or the DynamoDB document client, the AWS CLI requires explicit DynamoDB type descriptors (`{"S": "..."}`, `{"N": "..."}`, `{"BOOL": true}`) for every attribute in `put-item` operations. This is a common source of confusion for engineers transitioning from SDK-level abstractions to raw CLI usage.

### 5. `--query` is More Reliable than Manual JSON Parsing

Parsing `jq` or `grep` on full JSON responses in scripts is brittle. The native `--query` JMESPath filter built into AWS CLI is more reliable, readable, and portable across environments.

### 6. On-Demand Billing Eliminates an Entire Class of Operational Errors

Choosing `PAY_PER_REQUEST` removes the need to monitor consumed capacity, set CloudWatch alarms for throttling, and tune read/write capacity units. For internal tooling and DevOps workflows, this is almost always the correct billing mode choice.

### 7. Pre-Flight Checks Reduce Debugging Time

Running `aws --version`, `aws configure list`, and `aws dynamodb list-tables` before any provisioning work takes less than ten seconds and eliminates the three most common failure modes: missing CLI, misconfigured credentials, and wrong region.

---

## Troubleshooting Reference

| Error | Root Cause | Resolution |
|---|---|---|
| `command not found: aws` | AWS CLI not installed | Install from `https://aws.amazon.com/cli` |
| `Unable to locate credentials` | No credentials configured | Run `aws configure` with valid Access Key and Secret Key |
| `ResourceInUseException` | Table already exists | Run `aws dynamodb list-tables` to confirm, then delete or reuse |
| `ResourceNotFoundException` on `put-item` | Table not yet ACTIVE | Run `aws dynamodb wait table-exists` before inserting items |
| `ValidationException` | Missing type descriptors in `--item` | Use `{"S": "value"}` format for all attributes |
| `AccessDeniedException` | IAM policy does not include required action | Attach appropriate DynamoDB permissions to the IAM identity |
| `InvalidSignatureException` | System clock skew or wrong credentials | Sync system time and verify credentials are correct |
| JSON parse error on `--item` (Windows) | Shell quoting issue with single quotes | Use a `file://item.json` reference or PowerShell escape syntax |

---

## Repository Structure

```
.
|-- README.md
|-- screenshots/
|   |-- 01-aws-version.png
|   |-- 02-aws-configure-list.png
|   |-- 03-list-tables-empty.png
|   |-- 04-create-table-creating.png
|   |-- 05-wait-table-exists.png
|   |-- 06-describe-table-active.png
|   |-- 07-put-item-task1.png
|   |-- 08-put-item-task2.png
|   |-- 09-get-item-task1-verified.png
|   `-- 10-get-item-task2-verified.png
```

---

## Author

**Nautilus DevOps Team**
Region: `us-east-1`
Environment: AWS Cloud (Linux)

---

*Built and verified on Amazon Web Services. All commands executed and validated against live AWS infrastructure.*


<img width="1036" height="412" alt="Image" src="https://github.com/user-attachments/assets/f7466037-5d60-4cf1-b05d-6d1c8db0ec5b" />

<img width="1030" height="330" alt="Image" src="https://github.com/user-attachments/assets/2b710b78-38af-4812-a10a-343bc14e0871" />

<img width="1030" height="790" alt="Image" src="https://github.com/user-attachments/assets/79dddf71-f38e-4dd1-b9fb-1e66cb6279b9" />

<img width="1031" height="859" alt="Image" src="https://github.com/user-attachments/assets/4db339e5-ac7b-45f9-baf8-02174e4c6336" />

<img width="1032" height="238" alt="Image" src="https://github.com/user-attachments/assets/0f266a69-5775-426f-9e76-db805a0a04ae" />

<img width="1027" height="428" alt="Image" src="https://github.com/user-attachments/assets/93dced58-126c-471a-b3d1-5ae8d18d4837" />

<img width="1039" height="831" alt="Image" src="https://github.com/user-attachments/assets/1de9b9a1-4b48-4cd2-bc35-8dbb99a5db61" />
