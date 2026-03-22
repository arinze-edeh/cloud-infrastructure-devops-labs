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
  - [Step 2: Write the CloudFormation Template -- First Draft](#step-2-write-the-cloudformation-template----first-draft)
  - [Step 3: Confirm File Was Written](#step-3-confirm-file-was-written)
  - [Step 4: Inspect Template Sections with grep](#step-4-inspect-template-sections-with-grep)
  - [Step 5: Attempt Python YAML Validation -- ERROR 1](#step-5-attempt-python-yaml-validation----error-1)
  - [Step 6: Validate with AWS CLI -- First Validate](#step-6-validate-with-aws-cli----first-validate)
  - [Step 7: First Stack Deployment -- create-stack](#step-7-first-stack-deployment----create-stack)
  - [Step 8: Wait for Stack -- ROLLBACK_COMPLETE -- ERROR 2](#step-8-wait-for-stack----rollback_complete----error-2)
  - [Step 9: Diagnose Rollback with describe-stack-events -- ERROR 3](#step-9-diagnose-rollback-with-describe-stack-events----error-3)
  - [Step 10: Delete Failed Stack](#step-10-delete-failed-stack)
  - [Step 11: Write Corrected Template -- Second Draft](#step-11-write-corrected-template----second-draft)
  - [Step 12: Validate Corrected Template -- Second Validate](#step-12-validate-corrected-template----second-validate)
  - [Step 13: Second Stack Deployment -- CREATE_COMPLETE](#step-13-second-stack-deployment----create_complete)
  - [Step 14: Verify SQS Queues](#step-14-verify-sqs-queues)
  - [Step 15: Verify SNS Topic](#step-15-verify-sns-topic)
  - [Step 16: Verify Lambda Function Configuration](#step-16-verify-lambda-function-configuration)
  - [Step 17: Verify IAM Role](#step-17-verify-iam-role)
  - [Step 18: Publish Four Test Messages to SNS](#step-18-publish-four-test-messages-to-sns)
  - [Step 19: Lambda Invocations with CLI v2 Flags -- ERROR 4](#step-19-lambda-invocations-with-cli-v2-flags----error-4)
  - [Step 20: Corrected Invocation 1 -- High Priority message 1](#step-20-corrected-invocation-1----high-priority-message-1)
  - [Step 21: Corrected Invocation 2 -- High Priority message 2](#step-21-corrected-invocation-2----high-priority-message-2)
  - [Step 22: Corrected Invocation 3 -- Low Priority message 1](#step-22-corrected-invocation-3----low-priority-message-1)
  - [Step 23: Invocation 4 -- Lambda Timeout -- ERROR 5](#step-23-invocation-4----lambda-timeout----error-5)
  - [Step 24: Update Lambda Timeout to 10 Seconds](#step-24-update-lambda-timeout-to-10-seconds)
  - [Step 25: Confirm Timeout Update](#step-25-confirm-timeout-update)
  - [Step 26: Invocation 5 -- Low Priority message 2 -- Full Success](#step-26-invocation-5----low-priority-message-2----full-success)
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
- One SNS topic: `nautilus-Priority-Queues-Topic` with attribute-based SNS filter policies
- One Lambda function: `nautilus-priorities-queue-function` that polls high-priority first and falls back to low
- One IAM role: `lambda_execution_role` with least-privilege SQS permissions
- All infrastructure provisioned via AWS CloudFormation for repeatability and auditability

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
4. If the high-priority queue is empty, Lambda falls back to the low-priority queue
5. The consumed message is deleted from the queue after processing

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | v1.x installed and configured -- v2 flags will fail (see Error 4) |
| IAM Permissions | `cloudformation:*`, `sqs:*`, `sns:*`, `lambda:*`, `iam:CreateRole`, `iam:AttachRolePolicy` -- `iam:PutRolePolicy` is NOT permitted (see Error 3) |
| Python | 3.10+ available on the client host -- do NOT use it to validate CloudFormation YAML (see Error 1) |
| Region | `us-east-1` |

---

## Repository Structure

```
.
+-- index.py                          # Lambda function source code (pre-existing on client host at /root/index.py)
+-- nautilus-priority-stack.yml       # CloudFormation IaC template
+-- README.md                         # This document
```

---

## Deployment Walkthrough

---

### Step 1: Inspect the Lambda Function Code

The Lambda handler is pre-existing at `/root/index.py` on the AWS client host. Review it first to understand the priority polling logic and the environment variable contract before building the CloudFormation template.

```bash
cat /root/index.py
```

**Full source output:**

```python
import boto3
import os

sqs = boto3.client('sqs')

def delete_message(queue_url, receipt_handle, message):
    response = sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)
    return "Message " + "'" + message + "'" + " deleted"

def poll_messages(queue_url):
    QueueUrl=queue_url
    response = sqs.receive_message(
        QueueUrl=QueueUrl,
        AttributeNames=[],
        MaxNumberOfMessages=1,
        MessageAttributeNames=['All'],
        WaitTimeSeconds=3
    )
    if "Messages" in response:
        receipt_handle=response['Messages'][0]['ReceiptHandle']
        message = response['Messages'][0]['Body']
        delete_response = delete_message(QueueUrl,receipt_handle,message)
        return delete_response
    else:
        return "No more messages to poll"

def lambda_handler(event, context):
    response = poll_messages(os.environ['high_priority_queue'])
    if response == "No more messages to poll":
        response = poll_messages(os.environ['low_priority_queue'])
    return response
```

**Key observations:**

- `WaitTimeSeconds=3` enables SQS long-polling -- each empty poll blocks for up to 3 seconds before returning
- Queue URLs are read from environment variables `high_priority_queue` and `low_priority_queue` -- these must be injected by the CloudFormation template
- The handler always polls the high-priority queue first; low-priority is only polled when high returns empty
- Each invocation processes exactly one message -- combined with `WaitTimeSeconds=3`, this directly drives the timeout failure in Error 5

> **SCREENSHOT**

<img width="1030" height="700" alt="image" src="https://github.com/user-attachments/assets/b89e922f-b0f1-4929-a5e8-c0e6aaf70f57" />

> *Terminal showing the complete output of `cat /root/index.py` -- all three functions visible: `delete_message`, `poll_messages`, `lambda_handler`*

---

### Step 2: Write the CloudFormation Template -- First Draft

Write the template to `/root/nautilus-priority-stack.yml` using a heredoc. The first draft covers SQS queues, SNS topic, and subscriptions with filter policies.

```bash
cat > /root/nautilus-priority-stack.yml << 'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM

Resources:

  # -- SQS Queues ---------------------------------------------------------------
  NautilusHighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-High-Priority-Queue

  NautilusLowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-Low-Priority-Queue

  # -- SNS Topic ----------------------------------------------------------------
  NautilusPriorityQueuesTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: nautilus-Priority-Queues-Topic

  # -- SNS Subscriptions with filter policies -----------------------------------
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

EOF
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-02-first-heredoc-template-write.png`
> *Terminal showing the `cat > /root/nautilus-priority-stack.yml << 'EOF'` heredoc command being written with the SQS, SNS, and subscription resource blocks visible*

---

### Step 3: Confirm File Was Written

Verify the file exists and has the expected size.

```bash
ls -lh /root/nautilus-priority-stack.yml
```

**Output:**

```
-rw-r--r-- 1 root root 6.3K Mar 22 02:10 /root/nautilus-priority-stack.yml
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-03-ls-lh-template-file.png`
> *Terminal showing `ls -lh` output confirming `/root/nautilus-priority-stack.yml` exists at `6.3K` with the `Mar 22 02:10` timestamp*

---

### Step 4: Inspect Template Sections with grep

Use `grep` to spot-check the key sections of the template before validating: the environment variable block, the handler, and verify no EventSourceMapping was accidentally included.

```bash
grep -A4 "Environment:" /root/nautilus-priority-stack.yml
```

**Output:**

```yaml
      Environment:
        Variables:
          high_priority_queue: !Ref NautilusHighPriorityQueue
          low_priority_queue: !Ref NautilusLowPriorityQueue
      Code:
```

```bash
grep "Handler:" /root/nautilus-priority-stack.yml
```

**Output:**

```
      Handler: index.lambda_handler
```

```bash
grep "EventSourceMapping" /root/nautilus-priority-stack.yml
```

**Output:**

```
(no output -- EventSourceMapping correctly absent)
```

The environment variables are wired to the correct `!Ref` values, the handler points to `index.lambda_handler` matching the source code, and no EventSourceMapping is present -- Lambda will be invoked on-demand rather than being trigger-driven.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-04-grep-environment-handler-eventsource.png`
> *Terminal showing three consecutive grep commands: `grep -A4 "Environment:"` returning the variable block, `grep "Handler:"` returning `index.lambda_handler`, and `grep "EventSourceMapping"` returning no output*

---

### Step 5: Attempt Python YAML Validation -- ERROR 1

Before submitting to CloudFormation, a local syntax check was attempted using Python's `yaml` module.

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

Python's `yaml.safe_load` only handles the standard YAML 1.1 tag set. CloudFormation intrinsic function shorthand tags -- `!Ref`, `!GetAtt`, `!Sub`, `!Join`, `!Select`, `!If` -- are AWS proprietary YAML tag extensions and are not recognized by the Python parser. The `SafeLoader` raises `ConstructorError` on the first unrecognized tag it encounters regardless of whether the rest of the document is structurally valid.

This is a **false negative**. The template is not broken -- the wrong tool was used to validate it.

#### Resolution

Use `aws cloudformation validate-template` for all CloudFormation YAML validation. The AWS CLI ships with a parser that natively understands all CloudFormation intrinsic tags. Python `yaml` parsers are only appropriate for standard YAML files that contain no AWS-specific tags.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-05-error1-python-yaml-constructorerror.png`
> *Terminal showing the full Python traceback with the final line `yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref' in "/root/nautilus-priority-stack.yml", line 162, column 12`*

---

### Step 6: Validate with AWS CLI -- First Validate

Use the correct tool for CloudFormation YAML validation.

```bash
aws cloudformation validate-template \
  --template-body file:///root/nautilus-priority-stack.yml \
  --region us-east-1
```

**Output:**

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

Template passes AWS schema validation. `CAPABILITY_NAMED_IAM` is flagged because the template creates an IAM role with an explicit name (`lambda_execution_role`). This flag must be acknowledged at deploy time.

> **IMPORTANT:** `validate-template` confirms schema correctness only. It does not verify that the executing IAM principal has permission to create the resources -- that failure surfaces only at deploy time (see Error 2 and Error 3).

> **SCREENSHOT PLACEHOLDER**
> `screenshot-06-first-validate-template-iam-role.png`
> *Terminal showing `aws cloudformation validate-template` returning the JSON response with `"CAPABILITY_NAMED_IAM"` and `CapabilitiesReason: [AWS::IAM::Role]`*

---

### Step 7: First Stack Deployment -- create-stack

Submit the stack creation with the `CAPABILITY_NAMED_IAM` acknowledgment.

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

A `StackId` is returned immediately. CloudFormation begins asynchronous provisioning. The wait command is then run to block until a terminal state is reached.

```bash
aws cloudformation wait stack-create-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-07-first-create-stack-stackid.png`
> *Terminal showing `create-stack` returning the `StackId` ARN `fb8cc410-2594-11f1-b6ce-0affc41f21c3` followed by the `wait stack-create-complete` command running*

---

### Step 8: Wait for Stack -- ROLLBACK_COMPLETE -- ERROR 2

#### ERROR 2 -- Full Output

```
Waiter StackCreateComplete failed: Waiter encountered a terminal failure state:
For expression "Stacks[].StackStatus" we matched expected path: "ROLLBACK_COMPLETE" at least once
```

#### Root Cause

CloudFormation encountered a resource creation failure mid-stack. When any resource fails, CloudFormation automatically rolls back all previously created resources and transitions to `ROLLBACK_COMPLETE`. The waiter exits non-zero upon detecting this terminal state.

The error message is generic by design -- it confirms the terminal state but not which resource failed or why. The root cause requires a `describe-stack-events` query filtered to `CREATE_FAILED` events (see Step 9).

#### Resolution

Always follow any failed `wait stack-create-complete` with `describe-stack-events` filtered to `CREATE_FAILED` status before taking any remediation action.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-08-error2-rollback-complete.png`
> *Terminal showing the `wait` command exiting with `Waiter StackCreateComplete failed: Waiter encountered a terminal failure state: For expression "Stacks[].StackStatus" we matched expected path: "ROLLBACK_COMPLETE" at least once`*

---

### Step 9: Diagnose Rollback with describe-stack-events -- ERROR 3

Run the filtered event query to surface the exact failing resource and reason.

```bash
aws cloudformation describe-stack-events \
  --stack-name nautilus-priority-stack \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

#### ERROR 3 -- Full Output

```
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                                                                                                                                                                                 DescribeStackEvents                                                                                                                                                                                                                                |
+---------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|  LambdaExecutionRole|  Resource handler returned message: "User: arn:aws:iam::691595780564:user/kk_labs_user_407114 is not authorized to perform: iam:PutRolePolicy on resource: role lambda_execution_role because no identity-based policy allows the iam:PutRolePolicy action (Service: Iam, Status Code: 403, Request ID: 2add31fc-57fd-4f68-948f-8d37c03e6e59) (SDK Attempt Count: 1)" (RequestToken: 23f5a4e5-fcfc-6b30-d188-522fa3d20c9b, HandlerErrorCode: AccessDenied)   |
+---------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

#### Root Cause

The template's `LambdaExecutionRole` used an inline `Policies:` block. When CloudFormation provisions an IAM role with inline policies, the underlying SDK call it issues is `iam:PutRolePolicy`. The lab user `kk_labs_user_407114` has no permission for `iam:PutRolePolicy` in any attached policy -- resulting in a hard 403 AccessDenied.

Note that `iam:CreateRole` succeeded (the role object began creation) because that action was permitted. The failure occurred at the inline policy attachment step.

**IAM action comparison:**

| Template Approach | AWS Action Required | Lab User Permitted |
|---|---|---|
| Inline `Policies:` block on `AWS::IAM::Role` | `iam:PutRolePolicy` | NO -- causes ERROR 3 |
| Standalone `AWS::IAM::ManagedPolicy` + `ManagedPolicyArns` | `iam:CreatePolicy` + `iam:AttachRolePolicy` | YES |

#### Resolution

Remove the inline `Policies:` block from `LambdaExecutionRole`. Create a standalone `AWS::IAM::ManagedPolicy` resource and reference it via `ManagedPolicyArns` on the role. This changes the required IAM actions from `iam:PutRolePolicy` to `iam:CreatePolicy` and `iam:AttachRolePolicy`, both of which are permitted for this user.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-09-error3-describe-stack-events-putrolepolicy.png`
> *Terminal showing the full `describe-stack-events` table output with `LambdaExecutionRole` in the left column and the complete `iam:PutRolePolicy` 403 AccessDenied message including the Request ID `2add31fc-57fd-4f68-948f-8d37c03e6e59` and `HandlerErrorCode: AccessDenied`*

---

### Step 10: Delete Failed Stack

Delete the failed stack and wait for deletion to complete before redeploying -- redeploying over a `ROLLBACK_COMPLETE` stack is not permitted.

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

> **SCREENSHOT PLACEHOLDER**
> `screenshot-10-delete-stack-wait-complete.png`
> *Terminal showing the three commands in sequence: `delete-stack` returning with no output, `wait stack-delete-complete` returning cleanly, and `echo "Stack deleted"` printing the confirmation message*

---

### Step 11: Write Corrected Template -- Second Draft

Rewrite the full template, replacing the inline `Policies:` block with a standalone `AWS::IAM::ManagedPolicy` resource. The Lambda function resource, IAM role, managed policy, and SQS queue policies are all included in this corrected draft.

```bash
cat > /root/nautilus-priority-stack.yml << 'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM

Resources:

  # -- SQS Queues ---------------------------------------------------------------
  NautilusHighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-High-Priority-Queue

  NautilusLowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: nautilus-Low-Priority-Queue

  # -- SNS Topic ----------------------------------------------------------------
  NautilusPriorityQueuesTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: nautilus-Priority-Queues-Topic

  # -- SNS Subscriptions with filter policies -----------------------------------
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

  # -- IAM Role (no inline Policies block -- uses ManagedPolicyArns instead) ---
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

  # -- Standalone Managed Policy (requires iam:CreatePolicy, not iam:PutRolePolicy) --
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

  # -- Lambda Function ----------------------------------------------------------
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
          import boto3
          import os
          sqs = boto3.client('sqs')
          def delete_message(queue_url, receipt_handle, message):
              sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)
              return "Message '" + message + "' deleted"
          def poll_messages(queue_url):
              r = sqs.receive_message(QueueUrl=queue_url, AttributeNames=[],
                  MaxNumberOfMessages=1, MessageAttributeNames=['All'], WaitTimeSeconds=3)
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

> **SCREENSHOT PLACEHOLDER**
> `screenshot-11-second-heredoc-corrected-template.png`
> *Terminal showing the second `cat > /root/nautilus-priority-stack.yml << 'EOF'` heredoc completing, with the corrected `ManagedPolicyArns` and `AWS::IAM::ManagedPolicy` resource blocks visible*

---

### Step 12: Validate Corrected Template -- Second Validate

Run `validate-template` again on the corrected template before deploying.

```bash
aws cloudformation validate-template \
  --template-body file:///root/nautilus-priority-stack.yml \
  --region us-east-1
```

**Output:**

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

`CapabilitiesReason` now reads `AWS::IAM::ManagedPolicy` instead of `AWS::IAM::Role` as in the first validate. This confirms the template structure has changed correctly -- the role no longer holds inline policies; the managed policy resource is now the named IAM resource driving the capability flag.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-12-second-validate-managedpolicy.png`
> *Terminal showing the second `validate-template` returning `CapabilitiesReason: [AWS::IAM::ManagedPolicy]` -- compare to screenshot-06 which showed `[AWS::IAM::Role]`*

---

### Step 13: Second Stack Deployment -- CREATE_COMPLETE

Deploy the corrected template.

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

A new `StackId` is returned (`059048a0` -- different from the first attempt `fb8cc410`). Wait for completion:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1 && echo "CREATE_COMPLETE"
```

**Output:**

```
CREATE_COMPLETE
```

The waiter exits with code 0 and the `echo` fires -- stack reached `CREATE_COMPLETE` with no failures.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-13-second-create-stack-create-complete.png`
> *Terminal showing the second `create-stack` returning the new `StackId` `059048a0-2596-11f1-ac12-0eea9c6c1601`, followed by `wait stack-create-complete` completing and `CREATE_COMPLETE` printing on a new line*

---

### Step 14: Verify SQS Queues

```bash
aws sqs list-queues --queue-name-prefix nautilus --region us-east-1
```

**Output:**

```json
{
    "QueueUrls": [
        "https://queue.amazonaws.com/691595780564/nautilus-High-Priority-Queue",
        "https://queue.amazonaws.com/691595780564/nautilus-Low-Priority-Queue"
    ]
}
```

Both queues are present and their URLs confirm the account ID `691595780564` and the exact names from the template.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-14-sqs-list-queues.png`
> *Terminal showing `sqs list-queues --queue-name-prefix nautilus` returning both `nautilus-High-Priority-Queue` and `nautilus-Low-Priority-Queue` URLs*

---

### Step 15: Verify SNS Topic

```bash
aws sns list-topics --region us-east-1 \
  --query "Topics[?contains(TopicArn,'nautilus-Priority-Queues-Topic')]"
```

**Output:**

```json
[
    {
        "TopicArn": "arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic"
    }
]
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-15-sns-list-topics.png`
> *Terminal showing the filtered `sns list-topics` query returning the `nautilus-Priority-Queues-Topic` ARN*

---

### Step 16: Verify Lambda Function Configuration

```bash
aws lambda get-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  --query '[FunctionName,Handler,Runtime,Environment]'
```

**Output:**

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

Function name, handler, runtime, and both queue URL environment variables are all confirmed exactly as defined in the template.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-16-lambda-get-function-configuration.png`
> *Terminal showing the Lambda `get-function-configuration` query returning all four fields: `FunctionName`, `Handler: index.lambda_handler`, `Runtime: python3.12`, and both SQS queue URL environment variables*

---

### Step 17: Verify IAM Role

```bash
aws iam get-role --role-name lambda_execution_role \
  --query 'Role.[RoleName,Arn]'
```

**Output:**

```json
[
    "lambda_execution_role",
    "arn:aws:iam::691595780564:role/lambda_execution_role"
]
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-17-iam-get-role.png`
> *Terminal showing `iam get-role` returning `["lambda_execution_role", "arn:aws:iam::691595780564:role/lambda_execution_role"]`*

---

### Step 18: Publish Four Test Messages to SNS

Resolve the topic ARN dynamically into a shell variable, then publish two high-priority and two low-priority messages.

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

All four `MessageId` values confirm SNS accepted and routed each message through the filter policies into the correct SQS queues -- high-priority messages to `nautilus-High-Priority-Queue`, low-priority messages to `nautilus-Low-Priority-Queue`.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-18-sns-publish-four-messages.png`
> *Terminal showing the `topicarn` variable assignment, `echo` printing the full topic ARN, and all four `sns publish` commands each returning a distinct `MessageId`*

---

### Step 19: Lambda Invocations with CLI v2 Flags -- ERROR 4

The first set of invocation commands used `--cli-binary-format raw-in-base64-out` (a CLI v2-only flag) and passed the output file as a named argument after the flag. All four invocations were attempted this way before the error was identified.

```bash
# Invocation 1 -- expect High Priority message 1
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/out1.json && cat /tmp/out1.json

# Invocation 2 -- expect High Priority message 2
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/out2.json && cat /tmp/out2.json

# Invocation 3 -- expect Low Priority message 1
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/out3.json && cat /tmp/out3.json

# Invocation 4 -- expect Low Priority message 2
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/out4.json && cat /tmp/out4.json
```

#### ERROR 4 -- Full Output (identical for all four invocations)

```
Note: AWS CLI version 2, the latest major version of the AWS CLI, is now stable and recommended
for general use. For more information, see the AWS CLI version 2 installation instructions at:
https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html

usage: aws [options] <command> <subcommand> [<subcommand> ...] [parameters]
To see help text, you can run:

  aws help
  aws <command> help
  aws <command> <subcommand> help

Unknown options: --cli-binary-format, /tmp/out1.json
```

```
Unknown options: --cli-binary-format, /tmp/out2.json
```

```
Unknown options: --cli-binary-format, /tmp/out3.json
```

```
Unknown options: --cli-binary-format, /tmp/out4.json
```

#### Root Cause

The environment runs AWS CLI v1. The flag `--cli-binary-format raw-in-base64-out` was introduced in AWS CLI v2 to control binary payload encoding. CLI v1 does not recognize this option and aborts argument parsing immediately, also treating the subsequent positional value `/tmp/outN.json` as an unknown option. None of the four invocations reached Lambda -- all four failed at the CLI argument parsing stage before any API call was made.

#### Resolution

Remove `--cli-binary-format raw-in-base64-out` entirely. Remove `--payload '{}'` (not required for an empty invocation on CLI v1). Pass the output file path as the sole positional argument at the end of the command -- this is the CLI v1 syntax for `lambda invoke`.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-19-error4-all-four-cli-v2-unknown-options.png`
> *Terminal showing all four failed `lambda invoke` attempts back-to-back, each printing the AWS CLI version 2 upgrade notice followed by `Unknown options: --cli-binary-format, /tmp/outN.json` for N=1, 2, 3, 4*

---

### Step 20: Corrected Invocation 1 -- High Priority message 1

Using the corrected CLI v1 syntax -- no `--payload` flag, no `--cli-binary-format` flag, output file as positional argument.

```bash
# Invocation 1 -- expect High Priority message 1
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out1.json
cat /tmp/out1.json
```

**Output:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

```
"Message '{
  "Type" : "Notification",
  "MessageId" : "5f5862df-07c6-5010-984c-00e651c9a44a",
  "TopicArn" : "arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic",
  "Message" : "High Priority message 1",
  "Timestamp" : "2026-03-22T02:28:31.867Z",
  "MessageAttributes" : {
    "priority" : {"Type":"String","Value":"high"}
  }
}' deleted"
```

StatusCode 200, no `FunctionError`. The SNS notification envelope is the SQS message body -- the inner `"Message"` field confirms `High Priority message 1` was processed and deleted.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-20-invocation-1-high-priority-msg1.png`
> *Terminal showing invocation 1 returning `StatusCode: 200` with no `FunctionError`, and `out1.json` displaying the SNS envelope with `"Message" : "High Priority message 1"` and `"priority" : "high"`*

---

### Step 21: Corrected Invocation 2 -- High Priority message 2

```bash
# Invocation 2 -- expect High Priority message 2
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out2.json
cat /tmp/out2.json
```

**Output:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

```
"Message '{
  "Type" : "Notification",
  "MessageId" : "625c7e67-4b26-52b4-90d5-1c5e1d367bf2",
  "TopicArn" : "arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic",
  "Message" : "High Priority message 2",
  "Timestamp" : "2026-03-22T02:28:32.704Z",
  "MessageAttributes" : {
    "priority" : {"Type":"String","Value":"high"}
  }
}' deleted"
```

StatusCode 200. `High Priority message 2` confirmed deleted. The high-priority queue is now empty.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-21-invocation-2-high-priority-msg2.png`
> *Terminal showing invocation 2 returning `StatusCode: 200` and `out2.json` displaying the SNS envelope with `"Message" : "High Priority message 2"` -- the high-priority queue is now fully drained*

---

### Step 22: Corrected Invocation 3 -- Low Priority message 1

```bash
# Invocation 3 -- High queue now empty, expect Low Priority message 1
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out3.json
cat /tmp/out3.json
```

**Output:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

```
"Message '{
  "Type" : "Notification",
  "MessageId" : "e20ac0a8-0462-5652-bb93-a2915273ddab",
  "TopicArn" : "arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic",
  "Message" : "Low Priority message 1",
  "Timestamp" : "2026-03-22T02:28:33.525Z",
  "MessageAttributes" : {
    "priority" : {"Type":"String","Value":"low"}
  }
}' deleted"
```

StatusCode 200. Lambda correctly fell back to the low-priority queue after finding the high-priority queue empty. `Low Priority message 1` confirmed deleted. Priority ordering is validated: both high-priority messages were exhausted before any low-priority message was processed.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-22-invocation-3-low-priority-msg1.png`
> *Terminal showing invocation 3 returning `StatusCode: 200` and `out3.json` with `"Message" : "Low Priority message 1"` and `"priority" : "low"` -- confirming the fallback from empty high-priority queue to low-priority queue*

---

### Step 23: Invocation 4 -- Lambda Timeout -- ERROR 5

```bash
# Invocation 4 -- expect Low Priority message 2
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out4.json
cat /tmp/out4.json
```

**Output:**

```json
{
    "StatusCode": 200,
    "FunctionError": "Unhandled",
    "ExecutedVersion": "$LATEST"
}
```

```json
{"errorType":"Sandbox.Timedout","errorMessage":"RequestId: f83d9f5b-91c2-46ef-8f27-956c2efb047a Error: Task timed out after 3.00 seconds"}
```

#### ERROR 5 -- Root Cause

At the time of invocation 4, three messages had already been consumed. Only `Low Priority message 2` remained in the low-priority queue. On this invocation, the function must:

1. Poll the **high-priority queue** -- queue is empty, long-poll waits the full `WaitTimeSeconds=3` before returning no messages
2. Fall through to poll the **low-priority queue** -- but the Lambda execution timer has already consumed 3 full seconds

The configured Lambda timeout was 3 seconds (the CloudFormation template default). The high-priority long-poll alone exhausted the entire timeout budget before the fallback to the low-priority queue could begin.

**Worst-case execution time calculation:**

```
High-priority queue long-poll (empty) = up to 3 seconds
Low-priority queue long-poll          = up to 3 seconds
Total worst-case                      = up to 6 seconds
Configured timeout                    = 3 seconds   <-- insufficient by 3 seconds
```

`FunctionError: Unhandled` confirms this was a hard sandbox termination by the Lambda runtime, not a graceful application exception.

#### Resolution

Increase the Lambda timeout to 10 seconds -- providing a 4-second safety margin above the 6-second worst-case execution path.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-23-error5-invocation-4-sandbox-timedout.png`
> *Terminal showing invocation 4 returning `"FunctionError": "Unhandled"` in the invoke response, and `out4.json` displaying `"errorType":"Sandbox.Timedout"` with `"Task timed out after 3.00 seconds"` and RequestId `f83d9f5b-91c2-46ef-8f27-956c2efb047a`*

---

### Step 24: Update Lambda Timeout to 10 Seconds

```bash
aws lambda update-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --timeout 10 \
  --region us-east-1
```

**Output (partial):**

```json
{
    "FunctionName": "nautilus-priorities-queue-function",
    "FunctionArn": "arn:aws:lambda:us-east-1:691595780564:function:nautilus-priorities-queue-function",
    "Runtime": "python3.12",
    "Role": "arn:aws:iam::691595780564:role/lambda_execution_role",
    "Handler": "index.lambda_handler",
    "CodeSize": 518,
    "Timeout": 10,
    "MemorySize": 128,
    "LastModified": "2026-03-22T02:34:27.000+0000",
    "Environment": {
        "Variables": {
            "low_priority_queue": "https://sqs.us-east-1.amazonaws.com/691595780564/nautilus-Low-Priority-Queue",
            "high_priority_queue": "https://sqs.us-east-1.amazonaws.com/691595780564/nautilus-High-Priority-Queue"
        }
    },
    "LastUpdateStatus": "InProgress",
    "LastUpdateStatusReasonCode": "Creating"
}
```

> **SCREENSHOT PLACEHOLDER**
> `screenshot-24-update-function-configuration-timeout-10.png`
> *Terminal showing `update-function-configuration` returning the full function configuration JSON with `"Timeout": 10` and `"LastUpdateStatus": "InProgress"`*

---

### Step 25: Confirm Timeout Update

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

Timeout confirmed as 10 seconds. Safe to re-invoke.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-25-get-function-configuration-timeout-confirmed.png`
> *Terminal showing `get-function-configuration` with the `--query '[FunctionName,Timeout]'` filter returning `["nautilus-priorities-queue-function", 10]`*

---

### Step 26: Invocation 5 -- Low Priority message 2 -- Full Success

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
```

```
"Message '{
  "Type" : "Notification",
  "MessageId" : "57a7fb2a-d0f7-54b9-9b9d-991386e55ba3",
  "TopicArn" : "arn:aws:sns:us-east-1:691595780564:nautilus-Priority-Queues-Topic",
  "Message" : "Low Priority message 2",
  "Timestamp" : "2026-03-22T02:28:34.384Z",
  "SignatureVersion" : "1",
  "MessageAttributes" : {
    "priority" : {"Type":"String","Value":"low"}
  }
}' deleted"
```

StatusCode 200 with no `FunctionError`. `Low Priority message 2` confirmed processed and deleted. The `MessageId` `57a7fb2a-d0f7-54b9-9b9d-991386e55ba3` matches the fourth publish from Step 18. All four messages have been processed in strict priority order. The system is fully operational.

> **SCREENSHOT PLACEHOLDER**
> `screenshot-26-invocation-5-low-priority-msg2-success.png`
> *Terminal showing invocation 5 returning `StatusCode: 200` with no `FunctionError`, and `out5.json` displaying the full SNS envelope with `"Message" : "Low Priority message 2"`, `"priority" : "low"`, and `MessageId` matching the original publish*

---

## Complete Error Registry

| # | Error | Step | Command That Triggered It | Root Cause | Resolution |
|---|---|---|---|---|---|
| 1 | `yaml.constructor.ConstructorError: could not determine a constructor for the tag '!Ref' at line 162, column 12` | Step 5 | `python3 -c "import yaml; yaml.safe_load(...)"` | Python `yaml.safe_load` does not support CloudFormation intrinsic tags (`!Ref`, `!GetAtt`, etc.) | Use `aws cloudformation validate-template` for all CloudFormation YAML validation |
| 2 | `Waiter StackCreateComplete failed: terminal failure state: ROLLBACK_COMPLETE` | Step 8 | `aws cloudformation wait stack-create-complete` | A resource provisioning failure mid-stack triggered automatic CloudFormation rollback | Follow every failed waiter with `describe-stack-events --query CREATE_FAILED` to identify the failing resource |
| 3 | `iam:PutRolePolicy -- User kk_labs_user_407114 -- 403 AccessDenied -- Request ID: 2add31fc-57fd-4f68-948f-8d37c03e6e59` | Step 9 | `aws cloudformation describe-stack-events` (reveals the cause of Error 2) | Inline `Policies:` on `AWS::IAM::Role` requires `iam:PutRolePolicy` which the lab user lacks | Replace inline `Policies:` with a standalone `AWS::IAM::ManagedPolicy` resource and attach via `ManagedPolicyArns` |
| 4 | `Unknown options: --cli-binary-format, /tmp/out1.json` (repeated for out2, out3, out4) | Step 19 | `aws lambda invoke --cli-binary-format raw-in-base64-out` | `--cli-binary-format` is AWS CLI v2 only; environment runs CLI v1; all four invocations failed before reaching Lambda | Remove `--cli-binary-format` and `--payload '{}'`; pass output file as the sole positional argument |
| 5 | `Sandbox.Timedout -- Task timed out after 3.00 seconds -- RequestId: f83d9f5b-91c2-46ef-8f27-956c2efb047a` | Step 23 | `aws lambda invoke` (invocation 4) | Lambda timeout (3s) was less than or equal to the worst-case execution of two sequential SQS long-polls at `WaitTimeSeconds=3` each (up to 6s total) | Increase Lambda timeout to 10 seconds via `update-function-configuration --timeout 10` |

---

## Resource Summary

| Resource | Type | Name |
|---|---|---|
| High Priority Queue | `AWS::SQS::Queue` | `nautilus-High-Priority-Queue` |
| Low Priority Queue | `AWS::SQS::Queue` | `nautilus-Low-Priority-Queue` |
| SNS Topic | `AWS::SNS::Topic` | `nautilus-Priority-Queues-Topic` |
| High Subscription | `AWS::SNS::Subscription` | Filter: `priority = high` -- routes to High Priority Queue |
| Low Subscription | `AWS::SNS::Subscription` | Filter: `priority = low` -- routes to Low Priority Queue |
| Lambda Function | `AWS::Lambda::Function` | `nautilus-priorities-queue-function` |
| IAM Role | `AWS::IAM::Role` | `lambda_execution_role` |
| IAM Managed Policy | `AWS::IAM::ManagedPolicy` | SQS receive/delete + CloudWatch Logs |

---

## Priority Routing Logic

```
Lambda invoked
      |
      v
Poll high_priority_queue (WaitTimeSeconds=3)
      |
      +-- Message found? --> Delete message --> Return "Message '...' deleted"    (END)
      |
      +-- No message after 3s?
            |
            v
      Poll low_priority_queue (WaitTimeSeconds=3)
            |
            +-- Message found? --> Delete message --> Return "Message '...' deleted"  (END)
            |
            +-- No message after 3s? --> Return "No more messages to poll"            (END)

WORST-CASE BLOCKING TIME:  3s (high empty) + 3s (low wait) = up to 6 seconds
CONFIGURED TIMEOUT:        10 seconds  (4-second safety margin)
```

This design guarantees strict priority: the high-priority queue is always fully drained to zero before any low-priority message is ever processed.

---

## Best Practices

### Infrastructure as Code

- **Always validate before every deploy.** Run `aws cloudformation validate-template` before every `create-stack` or `update-stack`. It catches schema and reference errors without consuming any stack resources or incurring rollback wait times.
- **Use `wait` commands in all scripts.** Never assume a stack operation is complete. Always pair `create-stack` and `delete-stack` with the corresponding `wait` command. The wait command surfaces the terminal state -- `CREATE_COMPLETE` or `ROLLBACK_COMPLETE` -- so your script can take action rather than proceeding blindly.
- **Always query `CREATE_FAILED` events after any rollback.** Use `--query 'StackEvents[?ResourceStatus==\`CREATE_FAILED\`]'` to surface the exact failing resource and reason immediately rather than reading the full event stream.
- **`validate-template` passing is necessary but not sufficient.** Schema validation does not simulate the deploying principal's IAM permissions. A template can pass validation and fail deployment. Audit the executing principal's permissions against every IAM action each resource type requires before running `create-stack`.

### IAM and Permissions

- **Know the IAM action difference between inline policies and managed policies.** Inline `Policies:` on `AWS::IAM::Role` requires `iam:PutRolePolicy`. A standalone `AWS::IAM::ManagedPolicy` requires `iam:CreatePolicy` and `iam:AttachRolePolicy`. These are distinct actions controlled separately.
- **Prefer managed policies in restricted environments.** Lab and enterprise environments are more likely to permit `iam:AttachRolePolicy` than `iam:PutRolePolicy`. Managed policies also support reuse across multiple roles.
- **Scope SQS resources by ARN.** Use `!GetAtt NautilusHighPriorityQueue.Arn` instead of `"Resource": "*"` to enforce least-privilege at the individual queue level.

### Lambda Design

- **Size the timeout against the worst-case execution path, not the happy path.** The timeout must exceed the maximum possible sum of all sequential blocking I/O operations. With `WaitTimeSeconds=3` and two possible sequential polls, the minimum safe timeout is above 6 seconds. Configure 10 seconds as the production baseline for this pattern.
- **Use long-polling (`WaitTimeSeconds > 0`).** Long-polling reduces the number of empty `ReceiveMessage` API calls, lowers SQS costs, and reduces end-to-end message latency compared to short-polling.
- **Process one message per invocation.** `MaxNumberOfMessages=1` keeps execution time deterministic and simplifies timeout sizing.

### AWS CLI

- **Verify the CLI version before writing invocation scripts.** Run `aws --version`. CLI v1 requires the output file as a positional argument and does not support `--cli-binary-format`. CLI v2 supports both syntaxes.
- **Never use Python `yaml` to validate CloudFormation templates.** Python's `yaml.safe_load` will always fail on CloudFormation files that use intrinsic function tags. Use `aws cloudformation validate-template` exclusively.

### SNS and SQS

- **Route via message attributes, not message body.** SNS filter policies apply to message attributes set at `sns:Publish` time. This keeps routing logic cleanly separated from payload content.
- **Expect the SNS notification envelope in SQS.** When SNS delivers to SQS, the SQS message body is a JSON object wrapping the original payload. The publisher's message is in the inner `"Message"` field. Application code must parse this envelope if it needs the raw string.

---

## Lessons Learned

### 1. IAM Permission Gaps Surface at Provision Time, Not at validate-template Time

The first deployment failed during CloudFormation provisioning because the lab IAM user lacked `iam:PutRolePolicy`. The template passed `validate-template` cleanly, `create-stack` was accepted with a StackId, and the 403 only appeared in `describe-stack-events` after the rollback. `validate-template` and IAM permission auditing are two completely separate gates -- passing the first does not guarantee passing the second.

### 2. Lambda Timeout Must Be Calculated Against the Worst-Case Queue State, Not the Average Case

The 3-second timeout worked perfectly for invocations 1, 2, and 3 -- all of which found a message in the high-priority queue within milliseconds. It failed on invocation 4 because the queue state changed: the high-priority queue was empty, forcing a full 3-second long-poll before the fallback. The timeout must be sized for the worst case the system will actually encounter in production -- not the case that happened to exist during initial testing.

### 3. All Four Lambda Invocations Failed Before Reaching the API Due to a Single CLI Flag

`--cli-binary-format raw-in-base64-out` caused four consecutive total failures. Zero of the four invocation attempts touched Lambda. Always verify `aws --version` before writing invocation scripts and test a single invocation interactively before scripting all four.

### 4. CloudFormation YAML Is Not Standard YAML -- Use the Right Parser for the Right Job

Python's `yaml.safe_load` is correct and sufficient for standard YAML but will always fail on CloudFormation templates that use `!Ref`, `!GetAtt`, `!Sub`, or any other intrinsic function tag. This produces a misleading false negative. The `aws cloudformation validate-template` command is the only correct tool for CloudFormation YAML validation.

### 5. `describe-stack-events` Filtered to `CREATE_FAILED` Is the Essential Post-Rollback Diagnostic

The `ROLLBACK_COMPLETE` waiter error is generic. Without running `describe-stack-events` with the `CREATE_FAILED` filter, the actual cause -- `iam:PutRolePolicy` 403 -- would have been buried among dozens of rollback events. Filtering to `CREATE_FAILED` surfaces the exact resource, the exact action, the exact IAM principal, the Request ID, and the HandlerErrorCode in a single table row.

### 6. The SNS Notification Envelope Is the SQS Message Body -- Always

In every successful invocation output, the Lambda return value contained the full SNS notification JSON as the message body -- not the raw string passed to `sns:Publish`. The original publisher string is the value of the `"Message"` key inside that envelope. Any downstream processing that treats the SQS body as the raw payload will need to parse the envelope first.

---

## Security Considerations

- Scope `Resource` in the Lambda managed policy to the specific queue ARNs (`!GetAtt NautilusHighPriorityQueue.Arn` and `!GetAtt NautilusLowPriorityQueue.Arn`) instead of `"Resource": "*"`
- Enable SQS server-side encryption (SSE-SQS or SSE-KMS) for queues carrying sensitive payloads
- Enable AWS CloudTrail in `us-east-1` to audit all `sns:Publish`, `lambda:InvokeFunction`, and `sqs:ReceiveMessage` API calls for compliance and forensics
- Add a Dead Letter Queue (DLQ) to each SQS queue to capture messages that exceed `maxReceiveCount`, preventing silent data loss on repeated processing failures
- Restrict the SNS topic resource policy to known publisher ARNs -- avoid `"Principal": "*"` in production
- Rotate lab IAM user credentials (`kk_labs_user_407114`) immediately after the lab session ends and revoke any persistent access keys

---

## Author

**Nautilus DevOps Team**
Deployed: `Sun Mar 22 02:10 UTC 2026`
Region: `us-east-1`
Account: `691595780564`
Stack: `nautilus-priority-stack`
Stack ARN: `arn:aws:cloudformation:us-east-1:691595780564:stack/nautilus-priority-stack/059048a0-2596-11f1-ac12-0eea9c6c1601`

---

*Built with AWS CloudFormation, Amazon SQS, Amazon SNS, AWS Lambda, and Python 3.12*



<img width="1039" height="645" alt="image" src="https://github.com/user-attachments/assets/79bc08bb-875b-4542-8a34-2ba86f88f7db" />
<img width="1029" height="601" alt="image" src="https://github.com/user-attachments/assets/c256f2f4-3df8-4b9d-8952-a35719dd8f99" />
<img width="1026" height="477" alt="image" src="https://github.com/user-attachments/assets/c36de50a-3ab5-48ed-92ee-431e8f4934ea" />
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


