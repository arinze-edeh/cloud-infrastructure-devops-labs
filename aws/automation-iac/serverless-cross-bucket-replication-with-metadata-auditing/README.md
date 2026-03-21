# Automated S3-to-S3 File Replication Pipeline with Lambda and DynamoDB Audit Logging

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazon-aws)](https://aws.amazon.com)
[![Lambda](https://img.shields.io/badge/Lambda-Python%203.12-blue?logo=aws-lambda)](https://aws.amazon.com/lambda/)
[![DynamoDB](https://img.shields.io/badge/DynamoDB-NoSQL-green?logo=amazon-dynamodb)](https://aws.amazon.com/dynamodb/)
[![S3](https://img.shields.io/badge/S3-Object%20Storage-red?logo=amazon-s3)](https://aws.amazon.com/s3/)
[![IAM](https://img.shields.io/badge/IAM-Least%20Privilege-yellow)](https://aws.amazon.com/iam/)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen)]()

---

## Table of Contents

- [Business Problem](#business-problem)
- [Solution Architecture](#solution-architecture)
- [Infrastructure Components](#infrastructure-components)
- [Prerequisites](#prerequisites)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Phase 1: S3 Bucket Provisioning](#phase-1-s3-bucket-provisioning)
  - [Phase 2: IAM Role and Policy Configuration](#phase-2-iam-role-and-policy-configuration)
  - [Phase 3: Lambda Function Deployment](#phase-3-lambda-function-deployment)
  - [Phase 4: S3 Event Notification Wiring](#phase-4-s3-event-notification-wiring)
  - [Phase 5: DynamoDB Audit Table Provisioning](#phase-5-dynamodb-audit-table-provisioning)
  - [Phase 6: End-to-End Validation](#phase-6-end-to-end-validation)
- [Verification and Testing](#verification-and-testing)
- [CloudWatch Log Validation](#cloudwatch-log-validation)
- [DynamoDB Audit Record Schema](#dynamodb-audit-record-schema)
- [IAM Policy Reference](#iam-policy-reference)
- [Lambda Function Reference](#lambda-function-reference)
- [Best Practices Applied](#best-practices-applied)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [Cost Optimization Notes](#cost-optimization-notes)

---

## Business Problem

Enterprise DevOps teams managing data pipelines frequently require automated, auditable file replication between storage tiers. In many datacenter modernization and cloud migration workflows, files uploaded to a public-facing intake bucket must be automatically and securely propagated to a private storage zone, with a tamper-evident audit trail written to a durable data store.

**Without automation**, this process is manual, error-prone, lacks audit history, and introduces security gaps where sensitive files may remain unnecessarily exposed in a public bucket.

**This solution resolves:**

- Manual S3 copy operations with zero automation
- Absence of structured audit logging for file transfer events
- Inconsistent IAM permissions leading to over-privileged Lambda execution roles
- No observability into file movement between storage tiers
- Lack of event-driven architecture for real-time file processing

---

## Solution Architecture

This pipeline implements a fully event-driven, serverless architecture on AWS. When an object is uploaded to the public S3 bucket, an S3 event notification triggers an AWS Lambda function. The Lambda function copies the object to the private S3 bucket and writes a structured audit log entry to a DynamoDB table. All Lambda execution logs are streamed to AWS CloudWatch Logs for operational observability.

```
[User / Upload Client]
        |
        | PUT Object
        v
[S3: datacenter-public-14515]  <-- Public bucket (intake zone)
        |
        | s3:ObjectCreated:* event notification
        v
[Lambda: datacenter-copyfunction]  <-- Python 3.12 runtime
        |                  |
        | s3.copy_object   | dynamodb.put_item
        v                  v
[S3: datacenter-private-1666]   [DynamoDB: datacenter-S3CopyLogs]
  (Private, no public access)    (Audit log with LogID, timestamps,
                                  source/destination, status)
        |
        | All execution logs
        v
[CloudWatch Logs: /aws/lambda/datacenter-copyfunction]
```

---

## Infrastructure Components

| Component | Resource Name | Purpose |
|---|---|---|
| S3 Public Bucket | `datacenter-public-14515` | Intake zone for file uploads; public read access |
| S3 Private Bucket | `datacenter-private-1666` | Secure storage tier; all public access blocked |
| Lambda Function | `datacenter-copyfunction` | Event-driven copy handler (Python 3.12) |
| IAM Role | `lambda_execution_role` | Execution identity for Lambda with least-privilege permissions |
| IAM Policy | `lambda_s3_dynamodb_policy` | Scoped permissions for S3 read/write and DynamoDB writes |
| DynamoDB Table | `datacenter-S3CopyLogs` | Audit log store with `LogID` as partition key |
| CloudWatch Log Group | `/aws/lambda/datacenter-copyfunction` | Runtime log stream for Lambda execution |

---

## Prerequisites

- AWS CLI v1.x or v2.x configured (`aws configure`)
- IAM user with permissions to create S3 buckets, Lambda functions, IAM roles/policies, and DynamoDB tables
- Python 3.12 compatible Lambda deployment package
- `lambda-function.py` present in `/root/` on the AWS client host
- `sample.zip` present in `/root/` for end-to-end testing

Verify your identity and region before beginning:

```bash
aws sts get-caller-identity
aws configure get region
```

> **SCREENSHOT**

<img width="1038" height="494" alt="image" src="https://github.com/user-attachments/assets/2e98aa30-bbe9-43d7-96cc-80503db48643" />

> *Screenshot of `aws sts get-caller-identity` output confirming account ID and IAM user ARN*

---

## Step-by-Step Implementation

### Phase 1: S3 Bucket Provisioning

#### 1.1 Create the Public Intake Bucket

```bash
aws s3api create-bucket \
  --bucket datacenter-public-14515 \
  --region us-east-1
```

#### 1.2 Disable Public Access Block on Public Bucket

```bash
aws s3api put-public-access-block \
  --bucket datacenter-public-14515 \
  --public-access-block-configuration \
  "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

#### 1.3 Apply Public Read Bucket Policy

```bash
aws s3api put-bucket-policy \
  --bucket datacenter-public-14515 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "PublicReadGetObject",
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::datacenter-public-14515/*"
      }
    ]
  }'
```

#### 1.4 Verify Public Bucket Configuration

```bash
aws s3api get-bucket-policy --bucket datacenter-public-14515
aws s3api get-public-access-block --bucket datacenter-public-14515
```

> **SCREENSHOT**

<img width="1035" height="851" alt="image" src="https://github.com/user-attachments/assets/a44958ef-e155-42a1-a600-2decef758ccc" />

> *Screenshot showing the bucket policy JSON and public access block configuration confirming all four block flags are `false`*

#### 1.5 Create the Private Destination Bucket

```bash
aws s3api create-bucket \
  --bucket datacenter-private-1666 \
  --region us-east-1
```

#### 1.6 Enable All Public Access Blocks on Private Bucket

```bash
aws s3api put-public-access-block \
  --bucket datacenter-private-1666 \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

#### 1.7 Verify Private Bucket Lockdown

```bash
aws s3api get-public-access-block --bucket datacenter-private-1666
```

> **SCREENSHOT**

<img width="1040" height="466" alt="image" src="https://github.com/user-attachments/assets/d440dbce-f4ef-412e-9164-a1aec122dcce" />

> *Screenshot confirming all four public access block flags are `true` on the private bucket*

---

### Phase 2: IAM Role and Policy Configuration

#### 2.1 Create Lambda Trust Policy Document

```bash
cat > /tmp/trust-policy.json << 'EOF'
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
EOF
```

#### 2.2 Create the Lambda Execution Role

```bash
aws iam create-role \
  --role-name lambda_execution_role \
  --assume-role-policy-document file:///tmp/trust-policy.json
```

> **SCREENSHOT**

<img width="1037" height="805" alt="image" src="https://github.com/user-attachments/assets/e7becad1-1b85-4066-bc30-996957b439a0" />

> *Screenshot of the `create-role` response showing the Role ARN: `arn:aws:iam::604248583746:role/lambda_execution_role`*

#### 2.3 Create Scoped Permissions Policy

```bash
cat > /tmp/lambda-permissions-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3SourceRead",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::datacenter-public-14515/*"
    },
    {
      "Sid": "S3DestinationWrite",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::datacenter-private-1666/*"
    },
    {
      "Sid": "DynamoDBWrite",
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:604248583746:table/datacenter-S3CopyLogs"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
EOF
```

```bash
aws iam create-policy \
  --policy-name lambda_s3_dynamodb_policy \
  --policy-document file:///tmp/lambda-permissions-policy.json
```

#### 2.4 Attach Custom Policy to Role

```bash
aws iam attach-role-policy \
  --role-name lambda_execution_role \
  --policy-arn arn:aws:iam::604248583746:policy/lambda_s3_dynamodb_policy
```

#### 2.5 Attach AWS Managed Lambda Basic Execution Policy

```bash
aws iam attach-role-policy \
  --role-name lambda_execution_role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

#### 2.6 Verify Attached Policies

```bash
aws iam list-attached-role-policies --role-name lambda_execution_role
```

> **SCREENSHOTS**

<img width="1030" height="861" alt="image" src="https://github.com/user-attachments/assets/b5a44786-764f-4e34-aa02-85224eedabf6" />
<img width="1034" height="865" alt="image" src="https://github.com/user-attachments/assets/14ecea50-4c7c-45ad-9a32-8ee1ff83781e" />

> *Screenshots showing both `AWSLambdaBasicExecutionRole` and `lambda_s3_dynamodb_policy` listed as attached policies*

---

### Phase 3: Lambda Function Deployment

#### 3.1 Inject Environment-Specific Values into Lambda Source

Replace the placeholder values in `lambda-function.py` with your actual DynamoDB table name and private bucket name:

```bash
sed -i 's/REPLACE-WITH-YOUR-DYNAMODB-TABLE/datacenter-S3CopyLogs/g' /root/lambda-function.py
sed -i 's/REPLACE-WITH-YOUR-PRIVATE-BUCKET/datacenter-private-1666/g' /root/lambda-function.py
```

#### 3.2 Verify Substitutions Are Correct

```bash
cat /root/lambda-function.py
```

> **SCREENSHOTS**

<img width="1030" height="343" alt="image" src="https://github.com/user-attachments/assets/da52fc91-4278-44c2-a948-3be3319b8502" />
<img width="1031" height="839" alt="image" src="https://github.com/user-attachments/assets/f151b785-c140-409e-94db-a02cf7864194" />
<img width="1034" height="859" alt="image" src="https://github.com/user-attachments/assets/5f8e1237-d6c8-4901-9006-8043ad419bc8" />

> *Screenshots of `cat /root/lambda-function.py` output confirming `datacenter-S3CopyLogs` and `datacenter-private-1666` are correctly substituted at lines 8 and 14 respectively*

#### 3.3 Package Lambda Deployment Artifact

```bash
cd /root && zip lambda-function.zip lambda-function.py
ls -lh /root/lambda-function.zip
```

#### 3.4 Allow IAM Role Propagation

IAM role propagation in AWS can take up to 10 seconds. Always add a buffer before creating the Lambda function to avoid `InvalidParameterValueException` on role attachment:

```bash
sleep 10
```

#### 3.5 Deploy Lambda Function

```bash
aws lambda create-function \
  --function-name datacenter-copyfunction \
  --runtime python3.12 \
  --role arn:aws:iam::604248583746:role/lambda_execution_role \
  --handler lambda-function.lambda_handler \
  --zip-file fileb:///root/lambda-function.zip \
  --region us-east-1
```

> **SCREENSHOTS**

<img width="1028" height="864" alt="image" src="https://github.com/user-attachments/assets/2cdabfc0-1d83-4f2f-b71e-29f327ea7f4c" />
<img width="1033" height="854" alt="image" src="https://github.com/user-attachments/assets/7a749d7a-a45b-4662-849a-b202153d1387" />

> *Screenshots of the `create-function` response showing `"State": "Pending"` and the full Function ARN*

#### 3.6 Confirm Lambda is Active

```bash
aws lambda get-function \
  --function-name datacenter-copyfunction \
  --region us-east-1
```

> **SCREENSHOTS**

<img width="1037" height="848" alt="image" src="https://github.com/user-attachments/assets/81d40008-94bd-412c-84ef-07ca9e70ed5c" />
<img width="1032" height="865" alt="image" src="https://github.com/user-attachments/assets/019484de-de36-44ad-8864-f19b58026357" />

> *Screenshots of `get-function` output showing `"State": "Active"` and `"LastUpdateStatus": "Successful"`*

#### 3.7 Grant S3 Permission to Invoke Lambda

```bash
aws lambda add-permission \
  --function-name datacenter-copyfunction \
  --statement-id s3-trigger-permission \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::datacenter-public-14515 \
  --source-account 604248583746 \
  --region us-east-1
```

#### 3.8 Verify Lambda Resource-Based Policy

```bash
aws lambda get-policy \
  --function-name datacenter-copyfunction \
  --region us-east-1
```

> **SCREENSHOT**

<img width="1036" height="529" alt="image" src="https://github.com/user-attachments/assets/8599275d-558a-4a39-ad49-deaa1f33ae76" />

> *Screenshot of `get-policy` output confirming the S3 principal is allowed to invoke the function with the correct `ArnLike` condition on the source bucket ARN*

---

### Phase 4: S3 Event Notification Wiring

#### 4.1 Create Event Notification Configuration

```bash
cat > /tmp/s3-notification.json << 'EOF'
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:604248583746:function:datacenter-copyfunction",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}
EOF
```

#### 4.2 Apply Notification Configuration to Public Bucket

```bash
aws s3api put-bucket-notification-configuration \
  --bucket datacenter-public-14515 \
  --notification-configuration file:///tmp/s3-notification.json
```

#### 4.3 Verify Event Notification is Wired Correctly

```bash
aws s3api get-bucket-notification-configuration \
  --bucket datacenter-public-14515
```

> **SCREENSHOT**

<img width="1028" height="807" alt="image" src="https://github.com/user-attachments/assets/ff79f0f3-8d59-4ece-9369-3d9f4f8e5ef8" />

> *Screenshot confirming the `LambdaFunctionConfigurations` response shows the correct Lambda ARN and `s3:ObjectCreated:*` event type*

---

### Phase 5: DynamoDB Audit Table Provisioning

#### 5.1 Create DynamoDB Audit Log Table

```bash
aws dynamodb create-table \
  --table-name datacenter-S3CopyLogs \
  --attribute-definitions AttributeName=LogID,AttributeType=S \
  --key-schema AttributeName=LogID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

> **SCREENSHOT**

<img width="1031" height="853" alt="image" src="https://github.com/user-attachments/assets/d7cf8696-c45d-4e51-8d10-4058b7df78dd" />

> *Screenshot of the `create-table` response showing `"TableStatus": "CREATING"` and the Table ARN*

#### 5.2 Wait for Table to Become Active

```bash
aws dynamodb wait table-exists \
  --table-name datacenter-S3CopyLogs \
  --region us-east-1
```

#### 5.3 Confirm Table Status

```bash
aws dynamodb describe-table \
  --table-name datacenter-S3CopyLogs \
  --region us-east-1 \
  --query "Table.TableStatus"
```

Expected output:

```
"ACTIVE"
```

> **SCREENSHOT**

<img width="1032" height="597" alt="image" src="https://github.com/user-attachments/assets/7a0f958c-4cd4-49c2-bfc5-a58afd57d048" />

> *Screenshot showing `"ACTIVE"` status returned for the DynamoDB table*

---

### Phase 6: End-to-End Validation

#### 6.1 Upload Test File to Public Bucket

```bash
aws s3 cp /root/sample.zip s3://datacenter-public-14515/sample.zip
```

#### 6.2 Allow Lambda Execution Time

```bash
sleep 15
```

#### 6.3 Confirm File Replicated to Private Bucket

```bash
aws s3 ls s3://datacenter-private-1666/
```

Expected output:

```
2026-03-21 02:22:41   164 sample.zip
```

> **SCREENSHOT**

<img width="1029" height="404" alt="image" src="https://github.com/user-attachments/assets/0e32864b-f0c5-43af-a958-d7288c58684e" />

> *Screenshot of `aws s3 ls s3://datacenter-private-1666/` output confirming `sample.zip` is present with the correct size and timestamp*

#### 6.4 Scan DynamoDB for Audit Log Entry

```bash
aws dynamodb scan \
  --table-name datacenter-S3CopyLogs \
  --region us-east-1
```

Expected output structure:

```json
{
    "Items": [
        {
            "DestinationBucket": { "S": "datacenter-private-1666" },
            "LogID": { "S": "84c0c801-cb31-4002-9cf0-db4ed44efa25" },
            "Timestamp": { "S": "2026-03-21 02:22:40" },
            "Status": { "S": "Success" },
            "SourceBucket": { "S": "datacenter-public-14515" },
            "ObjectKey": { "S": "sample.zip" }
        }
    ],
    "Count": 1,
    "ScannedCount": 1
}
```

> **SCREENSHOT**

<img width="1030" height="842" alt="image" src="https://github.com/user-attachments/assets/0728106a-b1e3-4289-a6e6-f87e2a9f1ab0" />

> *Screenshot of the full DynamoDB scan response showing the audit record with `"Status": "Success"`, source/destination buckets, and the object key `sample.zip`*

---

## Verification and Testing

### Complete Validation Checklist

| Check | Command | Expected Result |
|---|---|---|
| Public bucket exists | `aws s3api head-bucket --bucket datacenter-public-14515` | HTTP 200 |
| Private bucket exists | `aws s3api head-bucket --bucket datacenter-private-1666` | HTTP 200 |
| Private bucket is locked | `aws s3api get-public-access-block --bucket datacenter-private-1666` | All flags `true` |
| Lambda is active | `aws lambda get-function --function-name datacenter-copyfunction` | `"State": "Active"` |
| S3 trigger configured | `aws s3api get-bucket-notification-configuration --bucket datacenter-public-14515` | Lambda ARN present |
| DynamoDB table active | `aws dynamodb describe-table --table-name datacenter-S3CopyLogs --query "Table.TableStatus"` | `"ACTIVE"` |
| File replicated | `aws s3 ls s3://datacenter-private-1666/` | `sample.zip` listed |
| Audit log written | `aws dynamodb scan --table-name datacenter-S3CopyLogs` | `"Status": "Success"` |

---

## CloudWatch Log Validation

Due to AWS CLI v1 not supporting `aws logs tail`, use `filter-log-events` to retrieve Lambda execution logs:

```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/datacenter-copyfunction \
  --region us-east-1 \
  --query "events[*].message" \
  --output text
```

**Expected log sequence:**

```
INIT_START Runtime Version: python:3.12.mainlinev2.v3
START RequestId: dbcd0015-198c-4c16-a13d-d8e6285dd9fd Version: $LATEST
[INFO] Source bucket: datacenter-public-14515, Object key: sample.zip
[INFO] Destination bucket: datacenter-private-1666
[INFO] Attempting to copy object from datacenter-public-14515/sample.zip to datacenter-private-1666/sample.zip
[INFO] File successfully copied from datacenter-public-14515/sample.zip to datacenter-private-1666/sample.zip
[INFO] Writing the following log entry to DynamoDB: { ... }
[INFO] Successfully wrote log entry to DynamoDB
END RequestId: dbcd0015-198c-4c16-a13d-d8e6285dd9fd
REPORT RequestId: dbcd0015-198c-4c16-a13d-d8e6285dd9fd  Duration: 541.47 ms  Billed Duration: 1053 ms  Memory Size: 128 MB  Max Memory Used: 95 MB  Init Duration: 511.15 ms
```

> **SCREENSHOT PLACEHOLDER**
> *Insert screenshot of the full CloudWatch `filter-log-events` output showing all INFO log lines, the DynamoDB JSON payload, and the REPORT line with duration and memory metrics*

---

## DynamoDB Audit Record Schema

| Attribute | Type | Description |
|---|---|---|
| `LogID` | String (PK) | UUID v4, unique per copy event |
| `SourceBucket` | String | Name of the public source bucket |
| `DestinationBucket` | String | Name of the private destination bucket |
| `ObjectKey` | String | S3 object key (file path) |
| `Timestamp` | String | UTC timestamp in `YYYY-MM-DD HH:MM:SS` format |
| `Status` | String | `Success` or `Failure` |
| `Error` | String | Error message (only present on `Failure` records) |

---

## IAM Policy Reference

### Custom Policy: `lambda_s3_dynamodb_policy`

This policy follows the principle of least privilege, granting only the minimum permissions required for each service interaction:

| Sid | Actions | Resource Scope |
|---|---|---|
| `S3SourceRead` | `s3:GetObject` | `datacenter-public-14515/*` only |
| `S3DestinationWrite` | `s3:PutObject` | `datacenter-private-1666/*` only |
| `DynamoDBWrite` | `dynamodb:PutItem` | `datacenter-S3CopyLogs` table only |
| `CloudWatchLogs` | `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` | All log groups (required for Lambda) |

### AWS Managed Policy: `AWSLambdaBasicExecutionRole`

Provides baseline CloudWatch Logs permissions required for all Lambda function executions. Attached in addition to the custom policy to follow the separation of concerns principle.

---

## Lambda Function Reference

**File:** `lambda-function.py`
**Runtime:** Python 3.12
**Handler:** `lambda-function.lambda_handler`
**Timeout:** 3 seconds (default)
**Memory:** 128 MB

**Core execution flow:**

1. Parse `event['Records'][0]['s3']` to extract `source_bucket` and `object_key`
2. Call `s3.copy_object()` to replicate from public to private bucket
3. Build a structured `log_entry` dict with UUID, timestamps, buckets, object key, and status
4. Call `table.put_item()` to persist audit record to DynamoDB
5. On any exception, attempt to write a `Failure` log entry to DynamoDB before raising

**Environment-injected values (set via `sed` substitution):**

```python
table = dynamodb.Table('datacenter-S3CopyLogs')
destination_bucket = "datacenter-private-1666"
```

---

## Best Practices Applied

### Security

- **Least Privilege IAM:** The custom policy grants only `s3:GetObject` on the source bucket and `s3:PutObject` on the destination bucket. No wildcard `s3:*` actions are used.
- **Resource-Level Conditions on Lambda Invocation:** The `add-permission` command includes both `--source-arn` and `--source-account` to prevent the confused deputy problem, ensuring only the intended bucket in the correct account can invoke the function.
- **Private Bucket Total Lockdown:** All four public access block flags (`BlockPublicAcls`, `IgnorePublicAcls`, `BlockPublicPolicy`, `RestrictPublicBuckets`) are set to `true` on the private bucket, providing defense in depth beyond just the absence of a bucket policy.
- **No Hardcoded Credentials:** The Lambda function uses `boto3` with the IAM execution role for all AWS API calls, never embedding access keys in application code.

### Reliability

- **IAM Propagation Sleep:** A 10-second `sleep` before `lambda create-function` prevents intermittent `InvalidParameterValueException` failures caused by IAM eventual consistency.
- **DynamoDB Error Logging:** The Lambda function has a nested try/catch pattern that attempts to write a `Failure` log entry to DynamoDB even when the S3 copy operation fails, preserving audit trail completeness.
- **`dynamodb wait table-exists`:** The deployment script waits for the DynamoDB table to reach `ACTIVE` state before proceeding to testing, avoiding race conditions.

### Observability

- **Structured CloudWatch Logging:** All Lambda log statements use `[INFO]` and `[ERROR]` prefixes with contextual data (bucket names, object keys, full DynamoDB payloads), making log filtering and metric extraction straightforward.
- **DynamoDB as a Durable Audit Store:** Using DynamoDB with `PAY_PER_REQUEST` billing provides a serverless, highly available, and infinitely scalable audit log without provisioned capacity management.
- **UUID as Log Partition Key:** Using `uuid4()` as the `LogID` partition key ensures uniform distribution across DynamoDB partitions and eliminates hot-key scenarios at scale.

### Operational Excellence

- **Infrastructure as CLI Scripts:** Every resource is provisioned via explicit, repeatable AWS CLI commands, enabling full reproducibility without manual console steps.
- **Verification at Each Phase:** Every provisioning step is followed by a read-back verification command (`get-bucket-policy`, `list-attached-role-policies`, `get-function`, `get-public-access-block`) to confirm the desired state before proceeding.
- **`sed` for Configuration Injection:** Using `sed -i` to replace placeholder values in the Lambda source file allows a single template function to be deployed to multiple environments without modifying source code directly.

---

## Lessons Learned

### 1. AWS CLI v1 Does Not Support `aws logs tail`

**Problem:** `aws logs tail` is an AWS CLI v2 command. Executing it on a system running AWS CLI v1.x returns `Invalid choice` error.

**Resolution:** Use `aws logs filter-log-events` with `--query "events[*].message"` as a direct replacement. This provides the same log content and is compatible with both CLI versions.

**Takeaway:** Always verify the AWS CLI major version at the start of any deployment. For greenfield projects, standardize on AWS CLI v2.

### 2. IAM Role Propagation Requires a Buffer Window

**Problem:** Immediately calling `lambda create-function` after `iam create-role` can result in `InvalidParameterValueException: The role defined for the function cannot be assumed by Lambda`, even though the role was successfully created.

**Resolution:** Insert a `sleep 10` between IAM role/policy creation and Lambda function deployment. For production automation pipelines, implement retry logic with exponential backoff instead of a fixed sleep.

**Takeaway:** IAM is an eventually consistent service. Any automation that creates an IAM role and immediately references it in another service must account for propagation delay.

### 3. Lambda `add-permission` Must Precede `put-bucket-notification-configuration`

**Problem:** If the S3 event notification is configured before Lambda has been granted permission to be invoked by S3, the `put-bucket-notification-configuration` call will fail with a permissions error.

**Resolution:** Always execute `lambda add-permission` before `s3api put-bucket-notification-configuration`. The correct sequence is: create function, add permission, then configure the notification.

**Takeaway:** S3 validates the Lambda invocation permission at the time the notification configuration is applied, not at invocation time.

### 4. `REPLACE-WITH-YOUR-*` Placeholder Pattern Enables Reusable Templates

**Problem:** Lambda function source code containing environment-specific resource names creates deployment friction and risks pushing incorrect configurations.

**Resolution:** Using named placeholders like `REPLACE-WITH-YOUR-DYNAMODB-TABLE` and `REPLACE-WITH-YOUR-PRIVATE-BUCKET` with `sed -i` substitution at deploy time cleanly separates configuration from code, enabling the same function template to be reused across environments (dev, staging, production) without source code modification.

**Takeaway:** Treat Lambda function templates as parameterized artifacts. Use environment variables or parameter injection patterns rather than embedding resource names directly.

### 5. DynamoDB `PAY_PER_REQUEST` is the Right Default for Event-Driven Audit Workloads

**Problem:** Pre-provisioning read/write capacity units for a DynamoDB table that receives intermittent, bursty Lambda writes leads to either over-provisioning (unnecessary cost) or throttling (lost audit records).

**Resolution:** `PAY_PER_REQUEST` billing mode automatically scales to match the exact request rate, eliminating throttling and removing the need to forecast capacity. For audit logging workloads where writes are event-driven and unpredictable, this is always the correct default.

**Takeaway:** Reserve provisioned capacity mode for DynamoDB tables with predictable, steady-state traffic. For Lambda-triggered writes, use `PAY_PER_REQUEST`.

### 6. Both `--source-arn` and `--source-account` Are Required in Lambda Permissions

**Problem:** Omitting `--source-account` from `lambda add-permission` when the principal is `s3.amazonaws.com` leaves the function vulnerable to the confused deputy attack, where any S3 bucket in any account could potentially trigger the function.

**Resolution:** Always specify both `--source-arn` (scoped to the specific bucket ARN) and `--source-account` (scoped to your account ID) when granting S3 invocation permissions to Lambda.

**Takeaway:** S3 as a principal does not carry account context by default. Explicitly binding both ARN and account ID is a required security control for Lambda functions triggered by S3.

---

## Troubleshooting

### Lambda function is not triggered after upload

- Verify the event notification is correctly configured: `aws s3api get-bucket-notification-configuration --bucket datacenter-public-14515`
- Verify Lambda has permission to be invoked by S3: `aws lambda get-policy --function-name datacenter-copyfunction`
- Check CloudWatch Logs for any invocation errors: `aws logs filter-log-events --log-group-name /aws/lambda/datacenter-copyfunction`

### File is not appearing in the private bucket

- Confirm the Lambda execution role has `s3:PutObject` on `datacenter-private-1666/*`
- Check CloudWatch Logs for `[ERROR]` lines indicating copy failures
- Verify the destination bucket name in the Lambda function matches the actual bucket: `grep destination_bucket /root/lambda-function.py`

### DynamoDB scan returns no items

- Confirm the table name in the Lambda function matches the actual table: `grep Table /root/lambda-function.py`
- Check CloudWatch Logs for `[ERROR] Failed to write error log entry to DynamoDB`
- Verify the execution role has `dynamodb:PutItem` on the correct table ARN

### `InvalidParameterValueException` on Lambda creation

- The IAM role has not fully propagated. Wait 10 to 15 seconds and retry the `create-function` command.

### `aws logs tail` returns `Invalid choice`

- You are running AWS CLI v1. Use `aws logs filter-log-events` as documented in [CloudWatch Log Validation](#cloudwatch-log-validation).

---

## Security Considerations

- **Rotate IAM credentials regularly** for the deploying IAM user (`kk_labs_user_659770`)
- **Enable S3 Server-Side Encryption (SSE-S3 or SSE-KMS)** on the private bucket for data at rest encryption
- **Enable S3 Versioning** on the private bucket to protect against accidental overwrites and deletions
- **Enable DynamoDB Point-in-Time Recovery (PITR)** on `datacenter-S3CopyLogs` for audit log durability
- **Restrict the `logs:*` CloudWatch permission** in production to the specific log group ARN pattern (`arn:aws:logs:us-east-1:604248583746:log-group:/aws/lambda/datacenter-copyfunction:*`)
- **Enable AWS CloudTrail** to audit all S3 API calls and Lambda invocations for compliance requirements
- **Consider VPC Lambda deployment** if the private S3 bucket will hold highly sensitive data, using VPC endpoints for S3 and DynamoDB to eliminate traffic traversal over the public internet

---

## Cost Optimization Notes

- Lambda charges apply only per invocation (100ms increments). At 128 MB memory and ~541 ms duration per copy event, the cost per invocation is negligible under AWS Free Tier and remains minimal at scale.
- DynamoDB `PAY_PER_REQUEST` eliminates idle capacity costs. Each `PutItem` write request unit (WRU) for the audit log entry is billed only when an object is copied.
- S3 charges apply for `PUT`, `COPY`, and `GET` requests as well as storage. Consider S3 Intelligent-Tiering on the private bucket for infrequently accessed files to reduce storage costs over time.
- CloudWatch Logs ingestion is charged per GB. The structured log format used in this solution is concise; no log compression or filtering optimizations are required at typical workload volumes.

---





<img width="1034" height="463" alt="image" src="https://github.com/user-attachments/assets/e73e2a88-eaaa-46c9-b05b-b6c1d4732dd4" />
<img width="1034" height="857" alt="image" src="https://github.com/user-attachments/assets/8426c5d2-3a89-4231-98f5-bae2a1317961" />
<img width="1029" height="857" alt="image" src="https://github.com/user-attachments/assets/9eca401e-7029-4759-bf3c-92cd00a1c33b" />

<img width="1031" height="271" alt="image" src="https://github.com/user-attachments/assets/4e9354e3-d650-4ef7-9e9b-4e84463d2532" />





<img width="1029" height="862" alt="image" src="https://github.com/user-attachments/assets/e6d09963-ad8f-4423-a1fd-e3363477217f" />



<img width="1038" height="869" alt="image" src="https://github.com/user-attachments/assets/198e32c6-9ef1-43f7-a7a0-e863ef4b1d1d" />
<img width="1031" height="863" alt="image" src="https://github.com/user-attachments/assets/2ba8d90f-3ed7-4148-9762-64d2c7387816" />
<img width="1036" height="554" alt="image" src="https://github.com/user-attachments/assets/ebe427af-1bca-4697-a1cf-595b1eebb24f" />
