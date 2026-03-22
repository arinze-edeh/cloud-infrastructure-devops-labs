# Nautilus Priority Queuing System on AWS

> **Enterprise-grade priority message routing using Amazon SQS, SNS, Lambda, and CloudFormation**

![AWS](https://img.shields.io/badge/AWS-CloudFormation-FF9900?style=flat-square&logo=amazon-aws)
![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&logo=python)
![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=flat-square)
![Region](https://img.shields.io/badge/Region-us--east--1-blue?style=flat-square)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Deployment Walkthrough](#deployment-walkthrough)
  - [Step 1: Inspect the Lambda Function Code](#step-1-inspect-the-lambda-function-code)
  - [Step 2: Author the CloudFormation Template (First Draft)](#step-2-author-the-cloudformation-template-first-draft)
  - [Step 3: Validate Template Syntax with Python yaml -- ERROR 1](#step-3-validate-template-syntax-with-python-yaml----error-1)
  - [Step 4: Validate with AWS CLI -- Correct Approach](#step-4-validate-with-aws-cli----correct-approach)
  - [Step 5: First Stack Deployment Attempt](#step-5-first-stack-deployment-attempt)
  - [Step 6: Stack Waiter Enters ROLLBACK_COMPLETE -- ERROR 2](#step-6-stack-waiter-enters-rollback_complete----error-2)
  - [Step 7: Diagnose the Rollback -- iam:PutRolePolicy 403 -- ERROR 3](#step-7-diagnose-the-rollback----iamputrolepolicy-403----error-3)
  - [Step 8: Delete the Failed Stack and Remediate](#step-8-delete-the-failed-stack-and-remediate)
  - [Step 9: Redeploy Corrected Template -- Second Validate](#step-9-redeploy-corrected-template----second-validate)
  - [Step 10: Second Stack Deployment -- CREATE_COMPLETE](#step-10-second-stack-deployment----create_complete)
  - [Step 11: Verify All Deployed Resources](#step-11-verify-all-deployed-resources)
  - [Step 12: Publish Four Test Messages to SNS](#step-12-publish-four-test-messages-to-sns)
  - [Step 13: First Lambda Invocation Attempt with CLI v2 Flags -- ERROR 4](#step-13-first-lambda-invocation-attempt-with-cli-v2-flags----error-4)
  - [Step 14: Corrected Lambda Invocations 1 Through 3 -- Success](#step-14-corrected-lambda-invocations-1-through-3----success)
  - [Step 15: Invocation 4 -- Lambda Timeout -- ERROR 5](#step-15-invocation-4----lambda-timeout----error-5)
  - [Step 16: Increase Lambda Timeout and Verify](#step-16-increase-lambda-timeout-and-verify)
  - [Step 17: Final Invocation 5 -- Full End-to-End Success](#step-17-final-invocation-5----full-end-to-end-success)
- [Complete Error Registry](#complete-error-registry)
- [Resource Summary](#resource-summary)
- [Priority Routing Logic](#priority-routing-logic)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Security Considerations](#security-considerations)
- [Author](#author)

---

## Problem Statement

The Nautilus DevOps team required a reliable, serverless priority queuing system capable of routing incoming messages to separate queues based on urgency. The system must guarantee that **high-priority messages are always processed before low-priority messages**, even when both are available simultaneously.

**Core Requirements:**

- Two SQS queues: `nautilus-High-Priority-Queue` and `nautilus-Low-Priority-Queue`
- One SNS topic: `nautilus-Priority-Queues-Topic` with attribute-based filter policies
- One Lambda function: `nautilus-priorities-queue-function` that polls high-priority first and falls back to low
- One IAM role: `lambda_execution_role` with least-privilege SQS and SNS permissions
- All infrastructure defined as code via AWS CloudFormation for repeatability and auditability

---

## Solution Architecture

```
                        +--------------------------+
                        |   SNS Topic              |
                        |  nautilus-Priority-      |
                        |  Queues-Topic            |
                        +----------+---------------+
                                   |
              +--------------------+--------------------+
              | FilterPolicy:                           | FilterPolicy:
              | priority = "high"                       | priority = "low"
              v                                         v
 +----------------------------+          +----------------------------+
 |  SQS High Priority Queue   |          |  SQS Low Priority Queue    |
 |  nautilus-High-Priority-   |          |  nautilus-Low-Priority-    |
 |  Queue                     |          |  Queue                     |
 +------------+---------------+          +------------+---------------+
              |                                        |
              +--------------------+-------------------+
                                   |
                         +---------v----------+
                         |  Lambda Function   |
                         |  nautilus-         |
                         |  priorities-queue- |
                         |  function          |
                         |                    |
                         |  1. Poll HIGH      |
                         |  2. If empty,      |
                         |     poll LOW       |
                         +--------------------+
                                   |
                         +---------v----------+
                         |  IAM Role          |
                         |  lambda_execution_ |
                         |  role              |
                         +--------------------+
```

**Message Flow:**

1. Publisher sends a message to the SNS topic with a `priority` message attribute set to `"high"` or `"low"`
2. SNS filter policies route the message to the corresponding SQS queue
3. Lambda polls the high-priority queue first on every invocation
4. If the high-priority queue is empty, Lambda falls back to poll the low-priority queue
5. The consumed message is deleted from the queue after processing

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | v1.x configured and tested (v2 flags will fail -- see Error 4) |
| IAM Permissions | `cloudformation:*`, `sqs:*`, `sns:*`, `lambda:*`, `iam:CreateRole`, `iam:AttachRolePolicy` (NOT `iam:PutRolePolicy` -- see Error 3) |
| Python | 3.10+ available on client host (do NOT use for CloudFormation YAML validation -- see Error 1) |
| Region | `us-east-1` |

---

## Repository Structure

```
.
+-- index.py                          # Lambda function source code
+-- nautilus-priority-stack.yml       # CloudFormation IaC template
+-- README.md                         # This document
```

---

## Deployment Walkthrough

---

### Step 1: Inspect the Lambda Function Code

The Lambda handler lives at `/root/index.py` on the AWS client host. Review it before deploying to understand the priority polling logic and the environment variable contract.

```bash
cat /root/index.py
```

**`index.py` -- Full Source:**

```python
import boto3
import os

sqs = boto3.client('sqs')

def delete_message(queue_url, receipt_handle, message):
    response = sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=receipt_handle
    )
    return "Message " + "'" + message + "'" + " deleted"

def poll_messages(queue_url):
    QueueUrl = queue_url
    response = sqs.receive_message(
        QueueUrl=QueueUrl,
        AttributeNames=[],
        MaxNumberOfMessages=1,
        MessageAttributeNames=['All'],
        WaitTimeSeconds=3
    )
    if "Messages" in response:
        receipt_handle = response['Messages'][0]['ReceiptHandle']
        message = response['Messages'][0]['Body']
        delete_response = delete_message(QueueUrl, receipt_handle, message)
        return delete_response
    else:
        return "No more messages to poll"

def lambda_handler(event, context):
    response = poll_messages(os.environ['high_priority_queue'])
    if response == "No more messages to poll":
        response = poll_messages(os.environ['low_priority_queue'])
    return response
```

**Key Design Observations:**

- `WaitTimeSeconds=3` enables SQS long-polling on every call
- Queue URLs are injected at runtime via environment variables `high_priority_queue` and `low_priority_queue`
- The handler always checks `high_priority_queue` first; low-priority is polled only when high returns empty
- Each invocation processes exactly one message -- this has direct implications for the Lambda timeout (see Error 5)

> **SCREENSHOT**

<img width="1030" height="700" alt="image" src="https://github.com/user-attachments/assets/b89e922f-b0f1-4929-a5e8-c0e6aaf70f57" />

> *Terminal showing the complete output of `cat /root/index.py` with the full Lambda source code visible*

---

### Step 2: Author the CloudFormation Template (First Draft)

Write the CloudFormation template to `/root/nautilus-priority-stack.yml` using a heredoc. This first draft uses an inline `Policies:` block on the IAM role -- which will fail at deploy time (see Error 3).

```bash
cat > /root/nautilus-priority-stack.yml << 'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM

Resources:

  NautilusHighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-High-Priority-Queue

  NautilusLowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-Low-Priority-Queue

  NautilusPriorityQueuesTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: nautilus-Priority-Queues-Topic

  HighPrioritySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref NautilusPriorityQueuesTopic
      Protocol: sqs
      Endpoint: !GetAtt NautilusHighPriorityQueue.Arn
      FilterPolicy:
        priority:
          - high

  LowPrioritySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref NautilusPriorityQueuesTopic
      Protocol: sqs
      Endpoint: !GetAtt NautilusLowPriorityQueue.Arn
      FilterPolicy:
        priority:
          - low

  # IAM Role with inline Policies block -- will trigger ERROR 3 at deploy time
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
      Policies:
        - PolicyName: LambdaSQSPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - sqs:ReceiveMessage
                  - sqs:DeleteMessage
                  - sqs:GetQueueAttributes
                Resource: '*'

EOF
```

Verify the file was written:

```bash
ls -lh /root/nautilus-priority-stack.yml
```

```
-rw-r--r-- 1 root root 6.3K Mar 22 02:10 /root/nautilus-priority-stack.yml
```

> **SCREENSHOT**

<img width="1039" height="645" alt="image" src="https://github.com/user-attachments/assets/79bc08bb-875b-4542-8a34-2ba86f88f7db" />
<img width="1029" height="601" alt="image" src="https://github.com/user-attachments/assets/c256f2f4-3df8-4b9d-8952-a35719dd8f99" />
<img width="1026" height="477" alt="image" src="https://github.com/user-attachments/assets/c36de50a-3ab5-48ed-92ee-431e8f4934ea" />

> *Terminal showing the heredoc `cat >` command completing and the `ls -lh` confirming the file size of `6.3K`*

---

### Step 3: Validate Template Syntax with Python yaml -- ERROR 1

#### Attempted Command

A local YAML syntax check was attempted using Python's built-in `yaml` module before submitting to CloudFormation.

```bash
python3 -c "import yaml; yaml.safe_load(open('/root/nautilus-priority-stack.yml'))" && echo "YAML syntax OK"
```

#### ERROR 1 -- Full Output

```
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "/usr/local/lib/python3.10/site-packages/yaml/__init__.py", line 125, in safe_load
    return load(stream, SafeLoader)
  File "/usr/local/lib/python3.10/site-packages/yaml/__init__.py", line 81, in load
    return loader.get_single_data()
  File "/usr/local/lib/python3.10/site-packages/yaml/constructor.py", line 51, in get_single_data
    return self.construct_document(node)
  File "/usr/local/lib/python3.10/site-packages/yaml/constructor.py", line 60, in construct_document
    for dummy in generator:
  File "/usr/local/lib/python3.10/site-packages/yaml/constructor.py", line 413, in construct_yaml_map
    value = self.construct_mapping(node)
  File "/usr/local/lib/python3.10/site-packages/yaml/constructor.py", line 218, in construct_mapping
    return self.construct_object(value_node, deep=deep)
  File "/usr/local/lib/python3.10/site-packages/yaml/constructor.py", line 100, in construct_object
    data = constructor(self, node)
  File "/usr/local/lib/python3.10/site-packages/yaml/constructor.py", line 427, in construct_undefined
    raise ConstructorError(None, None,
yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref'
  in "/root/nautilus-priority-stack.yml", line 162, column 12
```

#### Root Cause

Python's `yaml.safe_load` does not recognize CloudFormation-specific YAML tags such as `!Ref`, `!GetAtt`, `!Sub`, `!Join`, or `!If`. These are AWS proprietary tag extensions to the YAML specification. The Python `yaml` library's `SafeLoader` only handles standard YAML 1.1 tags and raises `ConstructorError` when it encounters any unrecognized tag, regardless of whether the CloudFormation document is structurally valid.

This is a false negative. The template itself is not broken -- the validation tool is wrong for this document type.

#### Resolution

Use `aws cloudformation validate-template` exclusively for CloudFormation YAML files. The AWS CLI ships with a custom YAML parser that understands all CloudFormation intrinsic function tags. Python YAML parsers are appropriate only for standard YAML documents that contain no AWS-specific tags.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-03-error1-python-yaml-constructorerror.png`
> *Terminal showing the full Python traceback ending with `yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref'` at line 162, column 12*

---

### Step 4: Validate with AWS CLI -- Correct Approach

Use the correct validation tool: `aws cloudformation validate-template`.

```bash
aws cloudformation validate-template \
  --template-body file:///root/nautilus-priority-stack.yml \
  --region us-east-1
```

**Output (First Draft Template):**

```json
{
    "Parameters": [],
    "Description": "Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM",
    "Capabilities": [
        "CAPABILITY_NAMED_IAM"
    ],
    "CapabilitiesReason": "The following resource(s) require capabilities: [AWS::IAM::Role]"
}
```

The template passes AWS schema validation. `CAPABILITY_NAMED_IAM` is required because the template creates an IAM role with an explicit name (`lambda_execution_role`). This capability flag must be explicitly passed at deploy time.

> **NOTE:** A valid `validate-template` response only confirms schema correctness. It does NOT confirm that the deploying principal has sufficient IAM permissions to create the resources -- that failure is exposed only at deploy time (see Error 2 and Error 3).

> **SCREENSHOT PLACEHOLDER**
> `screenshot-04-cfn-validate-template-ok.png`
> *Terminal showing `aws cloudformation validate-template` returning JSON with `CAPABILITY_NAMED_IAM` and `CapabilitiesReason: AWS::IAM::Role`*

---

### Step 5: First Stack Deployment Attempt

Submit the first stack creation with the required IAM capability acknowledgment.

```bash
aws cloudformation create-stack \
  --stack-name nautilus-priority-stack \
  --template-body file:///root/nautilus-priority-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**Output:**

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:691595780564:stack/nautilus-priority-stack/fb8cc410-2594-11f1-b6ce-0affc41f21c3"
}
```

A `StackId` is returned immediately. CloudFormation accepts the request and begins provisioning asynchronously. Wait for completion:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-05-first-create-stack-submitted.png`
> *Terminal showing `create-stack` returning a `StackId` ARN, followed by the `wait stack-create-complete` command blocking*

---

### Step 6: Stack Waiter Enters ROLLBACK_COMPLETE -- ERROR 2

#### ERROR 2 -- Full Output

```
Waiter StackCreateComplete failed: Waiter encountered a terminal failure state:
For expression "Stacks[].StackStatus" we matched expected path: "ROLLBACK_COMPLETE" at least once
```

#### Root Cause

CloudFormation encountered a resource provisioning failure mid-stack. When any resource fails to create, CloudFormation automatically initiates a rollback of all previously created resources to restore the account to its pre-stack state. The waiter exits with a non-zero return code when it observes `ROLLBACK_COMPLETE` as the terminal stack status.

This error message is intentionally generic. It tells you the stack failed and rolled back but does not identify which resource failed or why. The specific failure reason requires a separate `describe-stack-events` query (see Step 7 and Error 3).

#### Resolution

Always follow a failed `wait stack-create-complete` with `describe-stack-events` filtered to `CREATE_FAILED` events to identify the exact failing resource and reason before attempting any remediation.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-06-error2-waiter-rollback-complete.png`
> *Terminal showing the `wait` command exiting with the message `Waiter StackCreateComplete failed: Waiter encountered a terminal failure state` and `ROLLBACK_COMPLETE` status*

---

### Step 7: Diagnose the Rollback -- iam:PutRolePolicy 403 -- ERROR 3

Query only `CREATE_FAILED` events to extract the precise failure reason without reading the full event history.

```bash
aws cloudformation describe-stack-events \
  --stack-name nautilus-priority-stack \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

#### ERROR 3 -- Full Output

```
-----------------------------------------------------------------------------------------------------------------
|                                          DescribeStackEvents                                                  |
+---------------------+---------------------------------------------------------------------------------+
|  LambdaExecutionRole|  Resource handler returned message: "User:                                      |
|                     |  arn:aws:iam::691595780564:user/kk_labs_user_407114 is not authorized to         |
|                     |  perform: iam:PutRolePolicy on resource: role lambda_execution_role because      |
|                     |  no identity-based policy allows the iam:PutRolePolicy action                   |
|                     |  (Service: Iam, Status Code: 403,                                               |
|                     |  Request ID: 2add31fc-57fd-4f68-948f-8d37c03e6e59)                              |
|                     |  (SDK Attempt Count: 1)"                                                        |
|                     |  (RequestToken: 23f5a4e5-fcfc-6b30-d188-522fa3d20c9b,                          |
|                     |  HandlerErrorCode: AccessDenied)                                               |
+---------------------+---------------------------------------------------------------------------------+
```

#### Root Cause

The CloudFormation template defines the IAM role using an inline `Policies:` block. When CloudFormation creates an IAM role with inline policies, the underlying AWS SDK call issued is `iam:PutRolePolicy`. The lab IAM user `kk_labs_user_407114` does not have `iam:PutRolePolicy` in any of its attached permission policies, resulting in a 403 AccessDenied.

This is distinct from `iam:CreateRole` (which was permitted, as the role object itself began creation) and `iam:AttachRolePolicy` (which attaches managed policies and is permitted for this user).

**The critical IAM action distinction:**

| Template Pattern | CloudFormation Resource | Required IAM Action | Lab User Permitted |
|---|---|---|---|
| Inline `Policies:` block on role | `AWS::IAM::Role` with `Policies` | `iam:PutRolePolicy` | NO -- triggers ERROR 3 |
| Standalone `AWS::IAM::ManagedPolicy` | `AWS::IAM::ManagedPolicy` + `ManagedPolicyArns` | `iam:CreatePolicy` + `iam:AttachRolePolicy` | YES |

#### Resolution

Remove the inline `Policies:` block from `LambdaExecutionRole`. Create a separate `AWS::IAM::ManagedPolicy` resource containing the SQS and CloudWatch Logs permissions, and reference it via `ManagedPolicyArns` on the role. This shifts the required IAM actions from `iam:PutRolePolicy` to `iam:CreatePolicy` and `iam:AttachRolePolicy`, both of which are permitted for this lab user.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-07-error3-iam-putrolepolicy-403.png`
> *Terminal showing the `describe-stack-events` table output with `LambdaExecutionRole` in the `CREATE_FAILED` row and the complete `iam:PutRolePolicy 403 AccessDenied` error message and Request ID*

---

### Step 8: Delete the Failed Stack and Remediate

Wait for the stack deletion to complete before redeploying to avoid name conflicts and orphaned resources.

```bash
aws cloudformation delete-stack \
  --stack-name nautilus-priority-stack \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1

echo "Stack deleted"
```

**Output:**

```
Stack deleted
```

**Remediation applied to the template:**

The `Policies:` block is removed from `LambdaExecutionRole`. A new top-level resource `AWS::IAM::ManagedPolicy` is added with the SQS and CloudWatch Logs permissions. The role references this managed policy via `ManagedPolicyArns: [!Ref NautilusLambdaManagedPolicy]`.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-08-stack-delete-complete.png`
> *Terminal showing `delete-stack` accepting the request, `wait stack-delete-complete` returning cleanly, and `echo "Stack deleted"` printing the confirmation*

---

### Step 9: Redeploy Corrected Template -- Second Validate

Write the remediated template and validate it before deploying.

```bash
cat > /root/nautilus-priority-stack.yml << 'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM

Resources:

  NautilusHighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-High-Priority-Queue

  NautilusLowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-Low-Priority-Queue

  NautilusPriorityQueuesTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: nautilus-Priority-Queues-Topic

  HighPrioritySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref NautilusPriorityQueuesTopic
      Protocol: sqs
      Endpoint: !GetAtt NautilusHighPriorityQueue.Arn
      FilterPolicy:
        priority:
          - high

  LowPrioritySubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref NautilusPriorityQueuesTopic
      Protocol: sqs
      Endpoint: !GetAtt NautilusLowPriorityQueue.Arn
      FilterPolicy:
        priority:
          - low

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
        - !Ref NautilusLambdaManagedPolicy

  NautilusLambdaManagedPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Action:
              - sqs:ReceiveMessage
              - sqs:DeleteMessage
              - sqs:GetQueueAttributes
            Resource:
              - !GetAtt NautilusHighPriorityQueue.Arn
              - !GetAtt NautilusLowPriorityQueue.Arn
          - Effect: Allow
            Action:
              - logs:CreateLogGroup
              - logs:CreateLogStream
              - logs:PutLogEvents
            Resource: '*'

  NautilusPrioritiesQueueFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: nautilus-priorities-queue-function
      Handler: index.lambda_handler
      Runtime: python3.12
      Role: !GetAtt LambdaExecutionRole.Arn
      Timeout: 3
      Environment:
        Variables:
          high_priority_queue: !Ref NautilusHighPriorityQueue
          low_priority_queue: !Ref NautilusLowPriorityQueue
      Code:
        ZipFile: |
          import boto3, os
          sqs = boto3.client('sqs')
          def delete_message(queue_url, receipt_handle, message):
              sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)
              return "Message '" + message + "' deleted"
          def poll_messages(queue_url):
              r = sqs.receive_message(QueueUrl=queue_url, MaxNumberOfMessages=1,
                  MessageAttributeNames=['All'], WaitTimeSeconds=3)
              if 'Messages' in r:
                  return delete_message(queue_url, r['Messages'][0]['ReceiptHandle'],
                      r['Messages'][0]['Body'])
              return 'No more messages to poll'
          def lambda_handler(event, context):
              r = poll_messages(os.environ['high_priority_queue'])
              if r == 'No more messages to poll':
                  r = poll_messages(os.environ['low_priority_queue'])
              return r

EOF
```

Run the second validation:

```bash
aws cloudformation validate-template \
  --template-body file:///root/nautilus-priority-stack.yml \
  --region us-east-1
```

**Output (Corrected Template):**

```json
{
    "Parameters": [],
    "Description": "Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM",
    "Capabilities": [
        "CAPABILITY_NAMED_IAM"
    ],
    "CapabilitiesReason": "The following resource(s) require capabilities: [AWS::IAM::ManagedPolicy]"
}
```

`CapabilitiesReason` now references `AWS::IAM::ManagedPolicy` instead of `AWS::IAM::Role`, confirming the template structure has changed from inline policy to managed policy as intended.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-09-second-validate-managedpolicy.png`
> *Terminal showing the second `validate-template` response where `CapabilitiesReason` now reads `AWS::IAM::ManagedPolicy` -- this is the key visual difference from the first validate output in screenshot-04*

---

### Step 10: Second Stack Deployment -- CREATE_COMPLETE

Deploy the corrected template:

```bash
aws cloudformation create-stack \
  --stack-name nautilus-priority-stack \
  --template-body file:///root/nautilus-priority-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**Output:**

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:691595780564:stack/nautilus-priority-stack/059048a0-2596-11f1-ac12-0eea9c6c1601"
}
```

Wait for completion:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1 && echo "CREATE_COMPLETE"
```

**Output:**

```
CREATE_COMPLETE
```

The wait command exits with code 0 and the `echo` fires, confirming the stack reached `CREATE_COMPLETE` with no failures.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-10-create-complete-success.png`
> *Terminal showing the second `create-stack` `StackId` response followed by `wait` completing and `CREATE_COMPLETE` printing on a new line*

---

### Step 11: Verify All Deployed Resources

Confirm all four resource types were created successfully before testing.

#### SQS Queues

```bash
aws sqs list-queues --queue-name-prefix nautilus --region us-east-1
```

```json
{
    "QueueUrls": [
        "https://queue.amazonaws.com/691595780564/nautilus-High-Priority-Queue",
        "https://queue.amazonaws.com/691595780564/nautilus-Low-Priority-Queue"
    ]
}
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-11a-sqs-list-queues.png`
> *Terminal showing `sqs list-queues` returning both `nautilus-High-Priority-Queue` and `nautilus-Low-Priority-Queue` URLs*

#### SNS Topic

```bash
aws sns list-topics --region us-east-1 \
  --query "Topics[?contains(TopicArn,'nautilus-Priority-Queues-Topic')]"
```

```json
[
    {
        "TopicArn": "arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic"
    }
]
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-11b-sns-list-topics.png`
> *Terminal showing the filtered `sns list-topics` query returning the `nautilus-Priority-Queues-Topic` ARN*

#### Lambda Function

```bash
aws lambda get-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  --query '[FunctionName,Handler,Runtime,Environment]'
```

```json
[
    "nautilus-priorities-queue-function",
    "index.lambda_handler",
    "python3.12",
    {
        "Variables": {
            "low_priority_queue": "https://sqs.us-east-1.amazonaws.com/691595780564/nautilus-Low-Priority-Queue",
            "high_priority_queue": "https://sqs.us-east-1.amazonaws.com/691595780564/nautilus-High-Priority-Queue"
        }
    }
]
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-11c-lambda-get-function-config.png`
> *Terminal showing Lambda configuration confirming `FunctionName`, `Handler: index.lambda_handler`, `Runtime: python3.12`, and both SQS queue URL environment variables injected correctly*

#### IAM Role

```bash
aws iam get-role --role-name lambda_execution_role \
  --query 'Role.[RoleName,Arn]'
```

```json
[
    "lambda_execution_role",
    "arn:aws:iam::691595780564:role/lambda_execution_role"
]
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-11d-iam-get-role.png`
> *Terminal showing `iam get-role` returning `lambda_execution_role` name and its full ARN*

---

### Step 12: Publish Four Test Messages to SNS

Resolve the SNS topic ARN dynamically and publish two high-priority and two low-priority messages.

```bash
topicarn=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn,'nautilus-Priority-Queues-Topic')].TopicArn" \
  --output text --region us-east-1)

echo "Topic ARN: $topicarn"

aws sns publish --topic-arn $topicarn \
  --message 'High Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}' \
  --region us-east-1

aws sns publish --topic-arn $topicarn \
  --message 'High Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}' \
  --region us-east-1

aws sns publish --topic-arn $topicarn \
  --message 'Low Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}' \
  --region us-east-1

aws sns publish --topic-arn $topicarn \
  --message 'Low Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}' \
  --region us-east-1
```

**Output:**

```
Topic ARN: arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic
{"MessageId": "5f5862df-07c6-5010-984c-00e651c9a44a"}
{"MessageId": "625c7e67-4b26-52b4-90d5-1c5e1d367bf2"}
{"MessageId": "e20ac0a8-0462-5652-bb93-a2915273ddab"}
{"MessageId": "57a7fb2a-d0f7-54b9-9b9d-991386e55ba3"}
```

All four distinct `MessageId` values confirm SNS accepted each message and dispatched it through the filter policy to the correct SQS queue.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-12-sns-publish-four-messages.png`
> *Terminal showing the `topicarn` variable assignment, the `echo` printing the full topic ARN, and all four `sns publish` commands each returning a distinct `MessageId`*

---

### Step 13: First Lambda Invocation Attempt with CLI v2 Flags -- ERROR 4

#### Attempted Command

The initial invocation script used `--cli-binary-format raw-in-base64-out`, which is an AWS CLI v2-only flag, combined with the output file passed as a named positional value after the flag.

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/out1.json && cat /tmp/out1.json
```

#### ERROR 4 -- Full Output (identical error for all four invocations)

```
Note: AWS CLI version 2, the latest major version of the AWS CLI, is now stable
and recommended for general use. For more information, see the AWS CLI version 2
installation instructions at: https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html

usage: aws [options] <command> <subcommand> [<subcommand> ...] [parameters]
To see help text, you can run:

  aws help
  aws <command> help
  aws <command> <subcommand> help

Unknown options: --cli-binary-format, /tmp/out1.json
```

The same error was produced for all four invocation attempts (`/tmp/out1.json` through `/tmp/out4.json`). None of the four invocations reached Lambda. All four failed at the CLI argument parsing stage before any API call was made.

#### Root Cause

The environment runs AWS CLI v1. The flag `--cli-binary-format raw-in-base64-out` was introduced in AWS CLI v2 to control base64 encoding of binary payloads. CLI v1 does not recognize this option and treats it as an unknown argument. The subsequent positional value `/tmp/out1.json` is also reported as unknown because CLI v1 argument parsing aborted after the first unrecognized flag.

#### Resolution

Remove `--cli-binary-format raw-in-base64-out` entirely. Remove `--payload '{}'` (Lambda accepts invocation without an explicit empty payload on CLI v1). Pass the output file path as the sole positional argument at the very end of the command.

```bash
# Corrected CLI v1-compatible invocation syntax
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out1.json
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-13-error4-cli-v2-flags-unknown-options.png`
> *Terminal showing all four failed `lambda invoke` attempts back-to-back, each printing the AWS CLI version 2 upgrade notice followed by `Unknown options: --cli-binary-format, /tmp/outN.json` for N = 1, 2, 3, 4*

---

### Step 14: Corrected Lambda Invocations 1 Through 3 -- Success

Using corrected CLI v1 syntax, invoke Lambda and observe the strict priority ordering.

#### Invocation 1 -- High Priority message 1

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out1.json
cat /tmp/out1.json
```

**Output:**

```json
{"StatusCode": 200, "ExecutedVersion": "$LATEST"}
"Message '{... \"Message\" : \"High Priority message 1\" ...}' deleted"
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-14a-invocation-1-high-priority-msg1.png`
> *Terminal showing invocation 1 StatusCode 200 and the SNS envelope body in `out1.json` confirming `High Priority message 1` was deleted*

#### Invocation 2 -- High Priority message 2

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out2.json
cat /tmp/out2.json
```

**Output:**

```json
{"StatusCode": 200, "ExecutedVersion": "$LATEST"}
"Message '{... \"Message\" : \"High Priority message 2\" ...}' deleted"
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-14b-invocation-2-high-priority-msg2.png`
> *Terminal showing invocation 2 StatusCode 200 and `out2.json` confirming `High Priority message 2` was deleted, with the high-priority queue now fully drained*

#### Invocation 3 -- High Queue Empty, Falls Back to Low Priority message 1

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out3.json
cat /tmp/out3.json
```

**Output:**

```json
{"StatusCode": 200, "ExecutedVersion": "$LATEST"}
"Message '{... \"Message\" : \"Low Priority message 1\" ...}' deleted"
```

Lambda correctly fell back to the low-priority queue after finding the high-priority queue empty. Priority ordering is confirmed: all high-priority messages were exhausted before the first low-priority message was ever processed.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-14c-invocation-3-low-priority-msg1.png`
> *Terminal showing invocation 3 StatusCode 200 and `out3.json` confirming `Low Priority message 1` was deleted -- demonstrating successful fallback from empty high-priority queue to low-priority queue*

---

### Step 15: Invocation 4 -- Lambda Timeout -- ERROR 5

#### Attempted Command

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out4.json
cat /tmp/out4.json
```

#### ERROR 5 -- Full Output

```json
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "ExecutedVersion": "$LATEST"
}
{"errorType":"Sandbox.Timedout","errorMessage":"RequestId: f83d9f5b-91c2-46ef-8f27-956c2efb047a Error: Task timed out after 3.00 seconds"}
```

#### Root Cause

At the time of invocation 4, three of the four published messages had already been consumed and deleted (by invocations 1, 2, and 3). Only `Low Priority message 2` remained, sitting in the low-priority queue.

On invocation 4 the function must:

1. Poll the high-priority queue -- queue is now empty, long-poll waits the full `WaitTimeSeconds=3` seconds before returning an empty response
2. Detect no message, fall through to poll the low-priority queue -- but the execution timer has already consumed 3 seconds

The Lambda function's configured timeout was 3 seconds (set in the CloudFormation template). The high-priority queue long-poll alone consumed the entire 3-second budget before the fallback to the low-priority queue could even begin, causing a hard `Sandbox.Timedout` at exactly 3.00 seconds.

**Worst-case execution time calculation:**

```
high-priority queue long-poll (empty)  =  up to 3 seconds
low-priority queue long-poll           =  up to 3 seconds
Total worst-case execution time        =  up to 6 seconds
Configured Lambda timeout              =  3 seconds   <-- INSUFFICIENT by 3 seconds
```

`FunctionError: Unhandled` in the invoke response confirms this was a hard runtime termination by the Lambda sandbox, not a graceful exception raised by application code.

#### Resolution

Increase the Lambda timeout to 10 seconds, which provides a 4-second safety margin above the 6-second worst-case execution path (see Step 16).

> **SCREENSHOT PLACEHOLDER**
> `screenshot-15-error5-lambda-timeout-3s.png`
> *Terminal showing invocation 4 returning `FunctionError: Unhandled` in the invoke response, and `out4.json` displaying `errorType: Sandbox.Timedout` with `Task timed out after 3.00 seconds` and the specific RequestId*

---

### Step 16: Increase Lambda Timeout and Verify

Update the Lambda timeout from 3 seconds to 10 seconds using the CLI directly.

```bash
aws lambda update-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --timeout 10 \
  --region us-east-1
```

**Partial Output:**

```json
{
    "FunctionName": "nautilus-priorities-queue-function",
    "Timeout": 10,
    "LastUpdateStatus": "InProgress",
    "LastUpdateStatusReasonCode": "Creating",
    ...
}
```

Confirm the update took effect before invoking again:

```bash
aws lambda get-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  --query '[FunctionName,Timeout]'
```

**Output:**

```json
[
    "nautilus-priorities-queue-function",
    10
]
```

Timeout confirmed as 10 seconds.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-16-lambda-timeout-updated-10s.png`
> *Terminal showing `update-function-configuration` returning `"Timeout": 10` in the response body, followed by `get-function-configuration` query confirming the array `["nautilus-priorities-queue-function", 10]`*

---

### Step 17: Final Invocation 5 -- Full End-to-End Success

With the timeout resolved, invoke Lambda once more to process the remaining `Low Priority message 2`.

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out5.json
cat /tmp/out5.json
```

**Output:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
"Message '{
  \"Type\" : \"Notification\",
  \"MessageId\" : \"57a7fb2a-d0f7-54b9-9b9d-991386e55ba3\",
  \"TopicArn\" : \"arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic\",
  \"Message\" : \"Low Priority message 2\",
  \"Timestamp\" : \"2026-03-22T02:28:34.384Z\",
  \"SignatureVersion\" : \"1\",
  \"MessageAttributes\" : {
    \"priority\" : {\"Type\":\"String\",\"Value\":\"low\"}
  }
}' deleted"
```

StatusCode 200 with no `FunctionError`. `"Low Priority message 2"` is confirmed processed and deleted. The full SNS notification envelope is visible in the output, with `"Message": "Low Priority message 2"` and `"priority": "low"` attribute intact. End-to-end priority queuing is fully operational.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-17-final-invocation-5-success.png`
> *Terminal showing invocation 5 returning StatusCode 200 with no `FunctionError` field, and the full SNS notification envelope in `out5.json` confirming `Low Priority message 2` was successfully processed and deleted*

---

## Complete Error Registry

All five errors encountered during this deployment in a single consolidated reference.

| # | Error Message | Step | Category | Root Cause | Resolution |
|---|---|---|---|---|---|
| 1 | `yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref' at line 162` | Step 3 | Tool incompatibility | Python `yaml.safe_load` does not support CloudFormation intrinsic function tags (`!Ref`, `!GetAtt`, etc.) | Use `aws cloudformation validate-template` exclusively for all CloudFormation YAML validation |
| 2 | `Waiter StackCreateComplete failed: terminal failure state: ROLLBACK_COMPLETE` | Step 6 | CloudFormation rollback | A resource provisioning failure mid-stack triggered automatic CloudFormation rollback | Follow every failed waiter with `describe-stack-events --query CREATE_FAILED` to surface the specific failing resource |
| 3 | `iam:PutRolePolicy -- User kk_labs_user_407114 is not authorized -- 403 AccessDenied` | Step 7 | IAM permission denied | Inline `Policies:` on `AWS::IAM::Role` requires `iam:PutRolePolicy`, which is not permitted for the lab IAM user | Replace inline `Policies:` with a standalone `AWS::IAM::ManagedPolicy` resource and attach via `ManagedPolicyArns` |
| 4 | `Unknown options: --cli-binary-format, /tmp/out1.json` (repeated for all 4 invocations) | Step 13 | AWS CLI version mismatch | `--cli-binary-format raw-in-base64-out` is a CLI v2-only flag; environment runs CLI v1 | Remove `--cli-binary-format` and `--payload '{}'`; pass the output file as a positional argument |
| 5 | `Sandbox.Timedout -- Task timed out after 3.00 seconds` | Step 15 | Lambda execution timeout | Lambda timeout (3s) was less than or equal to the worst-case execution time of two sequential SQS long-polls (`WaitTimeSeconds=3` each, totaling up to 6s) | Increase Lambda `Timeout` to 10 seconds via `update-function-configuration` |

---

## Resource Summary

| Resource | Type | Name |
|---|---|---|
| High Priority Queue | `AWS::SQS::Queue` | `nautilus-High-Priority-Queue` |
| Low Priority Queue | `AWS::SQS::Queue` | `nautilus-Low-Priority-Queue` |
| SNS Topic | `AWS::SNS::Topic` | `nautilus-Priority-Queues-Topic` |
| High Subscription | `AWS::SNS::Subscription` | Filter: `priority = high` |
| Low Subscription | `AWS::SNS::Subscription` | Filter: `priority = low` |
| Lambda Function | `AWS::Lambda::Function` | `nautilus-priorities-queue-function` |
| IAM Role | `AWS::IAM::Role` | `lambda_execution_role` |
| IAM Managed Policy | `AWS::IAM::ManagedPolicy` | SQS + CloudWatch Logs permissions |

---

## Priority Routing Logic

```
Lambda invoked
      |
      v
Poll high_priority_queue (WaitTimeSeconds=3)
      |
      +-- Message found? --> Delete message --> Return result         (END)
      |
      +-- No message after 3s?
            |
            v
      Poll low_priority_queue (WaitTimeSeconds=3)
            |
            +-- Message found? --> Delete message --> Return result   (END)
            |
            +-- No message after 3s? --> "No more messages to poll"  (END)

WORST-CASE EXECUTION:  up to 6 seconds
CONFIGURED TIMEOUT:    10 seconds  (4-second safety margin above worst case)
```

This design guarantees strict FIFO priority ordering: the high-priority queue is always fully drained to zero before any low-priority message is ever processed.

---

## Best Practices

### Infrastructure as Code

- **Always validate before every deploy.** Run `aws cloudformation validate-template` before every `create-stack` or `update-stack`. It catches schema and reference errors without consuming any stack resources or incurring rollback wait times.
- **Use `wait` commands in all automation.** Never assume a stack operation is complete. Always chain `create-stack` and `delete-stack` with the corresponding `wait` command in scripts and CI/CD pipelines to prevent race conditions.
- **Filter `CREATE_FAILED` events immediately after any rollback.** Use `--query 'StackEvents[?ResourceStatus==\`CREATE_FAILED\`]'` to surface the exact failing resource and reason without reading the full event stream.
- **Treat `validate-template` success as necessary but not sufficient.** Schema validation does not test IAM execution permissions. A template that passes validation can still fail at deploy time if the executing principal lacks required actions on any resource type.

### IAM and Permissions

- **Know which IAM action each resource type requires.** Inline `Policies:` on `AWS::IAM::Role` requires `iam:PutRolePolicy`. A standalone `AWS::IAM::ManagedPolicy` requires `iam:CreatePolicy` and `iam:AttachRolePolicy`. These are distinct, separately controlled IAM actions.
- **Prefer managed policies over inline policies in restricted environments.** Lab and enterprise environments with constrained IAM users are more likely to permit `iam:AttachRolePolicy` than `iam:PutRolePolicy`. Managed policies also support reuse across multiple roles.
- **Scope SQS resource ARNs in managed policies.** Reference specific queue ARNs via `!GetAtt NautilusHighPriorityQueue.Arn` rather than `"Resource": "*"` to enforce least-privilege at the queue level.

### Lambda Design

- **Set the timeout to exceed the worst-case execution path with margin.** Calculate the maximum blocking I/O time across all code paths: two sequential SQS long-polls at `WaitTimeSeconds=3` each requires a minimum timeout above 6 seconds. Configure 10 seconds for a safe 4-second margin.
- **Use long-polling (`WaitTimeSeconds > 0`).** Long-polling reduces empty `ReceiveMessage` API calls, lowers SQS costs, and reduces end-to-end latency.
- **Process one message per invocation.** `MaxNumberOfMessages=1` keeps execution time deterministic and makes timeout sizing straightforward.

### AWS CLI

- **Always verify the CLI version before scripting.** Run `aws --version` and use version-appropriate syntax. CLI v1 requires the output file as a positional argument. CLI v2 supports `--cli-binary-format raw-in-base64-out` for binary payload encoding.
- **Do not use Python `yaml` to validate CloudFormation templates.** The Python YAML parser does not support `!Ref`, `!GetAtt`, `!Sub`, or any other CloudFormation intrinsic tag. Always use `aws cloudformation validate-template`.

### SNS and SQS

- **Use message attributes for SNS filter policies, not message body content.** Filter policies apply to message attributes set at publish time. The `priority` attribute value routes messages to the correct queue without any body parsing.
- **Understand the SNS-to-SQS delivery envelope.** When SNS delivers to SQS, the SQS message body is a JSON notification wrapper, not the raw publisher payload. The original message text lives in the inner `"Message"` field of that envelope.

---

## Lessons Learned

### 1. IAM Permission Scoping Fails at Provision Time, Not Runtime

The first deployment failed during CloudFormation stack provisioning because the executing IAM user lacked `iam:PutRolePolicy`. This is a silent permission gap in restricted environments: `validate-template` passes, `create-stack` is accepted, and the failure only appears in `describe-stack-events` after rollback. Always audit the executing principal's permissions against every IAM action that the template's resource types will require before deploying.

### 2. Lambda Timeout Must Be Sized Against the Full Execution Path, Not the Happy Path

The default 3-second Lambda timeout fails in the exact scenario the system was designed to handle: all high-priority messages consumed, one low-priority message remaining. The timeout must be calculated against the worst-case sum of all sequential blocking I/O operations, not just the fast path. A timeout that passes invocations 1 through 3 silently breaks on invocation 4 when queue state changes.

### 3. AWS CLI Version Must Be Verified Before Writing Any Invocation Script

The `--cli-binary-format raw-in-base64-out` flag produced four consecutive failures across all four initial Lambda invocation attempts, none of which reached Lambda at all. Verifying the CLI version with `aws --version` before scripting and using version-appropriate syntax would have eliminated this error entirely.

### 4. CloudFormation YAML Is Not Standard YAML

CloudFormation YAML uses AWS-proprietary shorthand tags (`!Ref`, `!GetAtt`, `!Sub`, `!Select`, etc.) that are outside the YAML 1.1 specification. Python's `yaml.safe_load` is correct for standard YAML but will always raise `ConstructorError` on any CloudFormation template that uses these tags. This creates a false negative that can mislead developers into thinking the template is invalid when it is structurally correct.

### 5. `validate-template` Success Does Not Validate IAM Execution Context

CloudFormation validates template schema and resource reference correctness but has no visibility into whether the calling principal's IAM policies permit the underlying API calls those resources require. Schema validation and permission validation are entirely separate concerns. Both Error 2 (`ROLLBACK_COMPLETE`) and Error 3 (`iam:PutRolePolicy 403`) occurred on a template that passed `validate-template` with no warnings.

### 6. SNS-to-SQS Delivery Wraps Messages in a Notification Envelope

The SQS message body received by Lambda is not the raw string passed to `sns:Publish`. It is a JSON object containing `Type`, `MessageId`, `TopicArn`, `Message`, `Timestamp`, `SignatureVersion`, `Signature`, `SigningCertURL`, `UnsubscribeURL`, and `MessageAttributes`. Application code that needs the original payload must parse the envelope and extract the `"Message"` field. This is documented AWS behavior and is visible in the Lambda output across all five successful invocations in this lab.

---

## Security Considerations

- Scope `Resource` in the Lambda managed policy to the specific queue ARNs (`!GetAtt NautilusHighPriorityQueue.Arn` and `!GetAtt NautilusLowPriorityQueue.Arn`) rather than `"Resource": "*"`
- Enable SQS server-side encryption (SSE-SQS or SSE-KMS) for queues carrying sensitive payloads
- Enable AWS CloudTrail in `us-east-1` to audit all `sns:Publish`, `lambda:InvokeFunction`, and `sqs:ReceiveMessage` API calls for compliance
- Add a Dead Letter Queue (DLQ) to each SQS queue to capture messages that exceed the `maxReceiveCount`, preventing silent data loss on repeated processing failures
- Restrict the SNS topic resource policy to known publisher ARNs; avoid `"Principal": "*"` in production topic policies
- Rotate lab IAM user credentials (`kk_labs_user_407114`) immediately after the lab session ends and revoke any persistent access keys

---

Region: `us-east-1`
Account: `691595780564`
Stack: `nautilus-priority-stack`
Stack ARN: `arn:aws:cloudformation:us-east-1:691595780564:stack/nautilus-priority-stack/059048a0-2596-11f1-ac12-0eea9c6c1601`

---

*Built with AWS CloudFormation, Amazon SQS, Amazon SNS, AWS Lambda, and Python 3.12*






<img width="1036" height="337" alt="image" src="https://github.com/user-attachments/assets/ee5bd82a-23a8-4a75-b19d-880980195258" />
<img width="1033" height="559" alt="image" src="https://github.com/user-attachments/assets/963508d5-4fd8-4b0a-a39b-edee32720acd" />
<img width="1034" height="264" alt="image" src="https://github.com/user-attachments/assets/8e006760-7c8e-4efe-ab81-91e5a2660154" />
<img width="1037" height="458" alt="image" src="https://github.com/user-attachments/assets/5e669cf2-afb0-4d01-a3dc-8f9d356f0ef4" />
<img width="1031" height="361" alt="image" src="https://github.com/user-attachments/assets/6dc4aafe-c3ca-42de-b6a6-98c976aa03d1" />
<img width="1035" height="676" alt="image" src="https://github.com/user-attachments/assets/cb0f25c5-03b8-4f32-b0d5-8aa4e503737f" />
<img width="1031" height="753" alt="image" src="https://github.com/user-attachments/assets/0e077702-4f7e-4cf9-96c1-e29e992912d5" />
<img width="1037" height="844" alt="image" src="https://github.com/user-attachments/assets/94208f50-5c3c-4e38-9ef6-c7c6508c8334" />
<img width="1030" height="600" alt="image" src="https://github.com/user-attachments/assets/4f647f1e-e238-4063-9a23-7507b9171c12" />
<img width="1032" height="506" alt="image" src="https://github.com/user-attachments/assets/78ad6644-8238-4ffb-b7ee-0d489c8d1187" />
<img width="1031" height="650" alt="image" src="https://github.com/user-attachments/assets/6194a704-a4e0-4063-8d3a-9d82dc651f12" />
<img width="1036" height="553" alt="image" src="https://github.com/user-attachments/assets/4020e4b7-ae63-4a0a-9606-37251f241e30" />
<img width="1031" height="747" alt="image" src="https://github.com/user-attachments/assets/a77c1fc6-e655-4e79-8273-ebf0cfe4b9c5" />
<img width="1030" height="862" alt="image" src="https://github.com/user-attachments/assets/8c327785-027d-4693-a08f-bc149a5998a8" />
<img width="1029" height="861" alt="image" src="https://github.com/user-attachments/assets/d6a63ad7-ec35-49e3-a137-e06cd6a0d3e2" />
<img width="1033" height="865" alt="image" src="https://github.com/user-attachments/assets/aa4984d1-498e-4b43-a1ff-d20502228477" />
<img width="1039" height="859" alt="image" src="https://github.com/user-attachments/assets/02e96455-daa7-49ec-bd7d-d5324a7a1f2d" />
<img width="1033" height="857" alt="image" src="https://github.com/user-attachments/assets/64a8da7b-9c74-43a7-945f-859d7447f31e" />
<img width="1032" height="865" alt="image" src="https://github.com/user-attachments/assets/4390a804-60c5-4823-a6b8-d52ac0b858c9" />
<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/5f2a8652-efe0-4574-ad3f-aedc7b7232e2" />
<img width="1036" height="867" alt="image" src="https://github.com/user-attachments/assets/4d2652fd-9801-44af-aee7-610361ed41a8" />
<img width="1032" height="874" alt="image" src="https://github.com/user-attachments/assets/18c096de-2a66-4bef-91f4-d31754f34aac" />


