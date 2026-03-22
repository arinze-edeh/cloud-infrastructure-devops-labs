# Nautilus Priority Queuing System -- AWS SQS, SNS, Lambda, and CloudFormation

[![AWS](https://img.shields.io/badge/AWS-CloudFormation-orange?logo=amazon-aws)](https://aws.amazon.com/cloudformation/)
[![SQS](https://img.shields.io/badge/AWS-SQS-yellow?logo=amazon-aws)](https://aws.amazon.com/sqs/)
[![SNS](https://img.shields.io/badge/AWS-SNS-pink?logo=amazon-aws)](https://aws.amazon.com/sns/)
[![Lambda](https://img.shields.io/badge/AWS-Lambda-orange?logo=amazon-aws)](https://aws.amazon.com/lambda/)
[![IAM](https://img.shields.io/badge/AWS-IAM-red?logo=amazon-aws)](https://aws.amazon.com/iam/)
[![Region](https://img.shields.io/badge/Region-us--east--1-blue)](https://aws.amazon.com/about-aws/global-infrastructure/regions_az/)
[![Status](https://img.shields.io/badge/Stack-CREATE__COMPLETE-brightgreen)]()

---

## Table of Contents

* [Overview](#overview)
* [Architecture](#architecture)
* [Problem Statement](#problem-statement)
* [Prerequisites](#prerequisites)
* [Repository Structure](#repository-structure)
* [CloudFormation Template Design](#cloudformation-template-design)
* [Step-by-Step Implementation](#step-by-step-implementation)
  * [Phase 1 -- Inspect Lambda Source Code](#phase-1----inspect-lambda-source-code)
  * [Phase 2 -- Author the CloudFormation Template](#phase-2----author-the-cloudformation-template)
  * [Phase 3 -- Validate the Template](#phase-3----validate-the-template)
  * [Phase 4 -- Deploy the Stack](#phase-4----deploy-the-stack)
  * [Phase 5 -- Verify All Resources](#phase-5----verify-all-resources)
  * [Phase 6 -- Test Priority Queuing Behavior](#phase-6----test-priority-queuing-behavior)
* [Problems Encountered and Resolutions](#problems-encountered-and-resolutions)
* [Test Results](#test-results)
* [Best Practices](#best-practices)
* [Lessons Learned](#lessons-learned)
* [Resource Cleanup](#resource-cleanup)

---

## Overview

This project implements an **enterprise-grade priority message queuing system** on AWS for the Nautilus DevOps team. The system routes messages with different priorities through dedicated Amazon SQS queues via an SNS fan-out pattern, with an AWS Lambda function that enforces strict priority-first consumption logic.

The entire infrastructure is provisioned as **Infrastructure as Code (IaC)** using a single AWS CloudFormation template, deployed from the AWS client host.

**Key outcome:** High-priority messages are always consumed and processed before any low-priority messages, regardless of arrival order.

---

## Architecture

```
                        +------------------------------+
                        |   Publisher (SNS Publish)    |
                        |  MessageAttribute: priority  |
                        |   "high" | "low"             |
                        +-------------+----------------+
                                      |
                                      v
                        +------------------------------+
                        |   nautilus-Priority-         |
                        |   Queues-Topic (SNS)         |
                        +-------+----------+-----------+
                                |          |
              FilterPolicy:     |          |     FilterPolicy:
              priority = high   |          |     priority = low
                                |          |
                                v          v
                +--------------------------+  +-------------------------+
                | nautilus-High-Priority-  |  | nautilus-Low-Priority-  |
                | Queue (SQS)              |  | Queue (SQS)             |
                +--------------------------+  +-------------------------+
                                |                        |
                                +----------+-------------+
                                           |
                              (Lambda polls High first)
                                           |
                                           v
                        +------------------------------+
                        | nautilus-priorities-queue-   |
                        | function (Lambda py3.12)     |
                        |                              |
                        |  1. Poll High Priority Queue |
                        |  2. If empty -> Poll Low     |
                        |  3. Delete processed message |
                        +------------------------------+
                                           |
                              IAM Role: lambda_execution_role
                              Permissions: SQS R/W, SNS, CloudWatch Logs
```

---

## Problem Statement

The Nautilus DevOps team required a priority-aware message processing system with the following hard requirements:

| Requirement | Detail |
|---|---|
| Infrastructure as Code | CloudFormation template at `/root/nautilus-priority-stack.yml` |
| Stack Name | `nautilus-priority-stack` |
| High Priority Queue | `nautilus-High-Priority-Queue` (SQS) |
| Low Priority Queue | `nautilus-Low-Priority-Queue` (SQS) |
| SNS Topic | `nautilus-Priority-Queues-Topic` |
| Lambda Function | `nautilus-priorities-queue-function` (code from `/root/index.py`) |
| IAM Role | `lambda_execution_role` |
| Region | `us-east-1` |
| Behavior | High-priority messages must always be processed first |

---

## Prerequisites

* AWS CLI v1 installed and configured on the AWS client host
* IAM user with permissions to deploy CloudFormation, create SQS, SNS, Lambda, and IAM resources
* Lambda source code available at `/root/index.py` on the client host
* Target AWS region: `us-east-1`

---

## Repository Structure

```
.
|-- README.md                        # This document
|-- nautilus-priority-stack.yml      # CloudFormation IaC template (deployed to /root/)
|-- index.py                         # Lambda function source code (pre-provided at /root/)
```

---

## CloudFormation Template Design

### Resources Provisioned

| Logical ID | Type | Physical Name |
|---|---|---|
| `NautilusHighPriorityQueue` | `AWS::SQS::Queue` | `nautilus-High-Priority-Queue` |
| `NautilusLowPriorityQueue` | `AWS::SQS::Queue` | `nautilus-Low-Priority-Queue` |
| `NautilusPriorityQueuesTopic` | `AWS::SNS::Topic` | `nautilus-Priority-Queues-Topic` |
| `HighPrioritySubscription` | `AWS::SNS::Subscription` | SNS to High Queue (filter: `priority=high`) |
| `LowPrioritySubscription` | `AWS::SNS::Subscription` | SNS to Low Queue (filter: `priority=low`) |
| `HighPriorityQueuePolicy` | `AWS::SQS::QueuePolicy` | Allows SNS to write to High Queue |
| `LowPriorityQueuePolicy` | `AWS::SQS::QueuePolicy` | Allows SNS to write to Low Queue |
| `LambdaSQSSNSManagedPolicy` | `AWS::IAM::ManagedPolicy` | Grants Lambda SQS/SNS/CloudWatch access |
| `LambdaExecutionRole` | `AWS::IAM::Role` | `lambda_execution_role` |
| `NautilusPrioritiesQueueFunction` | `AWS::Lambda::Function` | `nautilus-priorities-queue-function` |

### Critical Design Decisions

**1. No EventSourceMapping**
The Lambda function uses `sqs.receive_message()` with `WaitTimeSeconds=3` directly inside the handler -- it is NOT event-driven. Adding an `EventSourceMapping` would cause a conflict and is architecturally incorrect for this pattern.

**2. Environment Variables Required**
The Lambda code reads queue URLs from `os.environ['high_priority_queue']` and `os.environ['low_priority_queue']`. These must be injected via the `Environment.Variables` block using `!Ref` to the SQS queue resources.

**3. ManagedPolicy Instead of Inline Policy**
The lab IAM user lacks `iam:PutRolePolicy` permission. Using `AWS::IAM::ManagedPolicy` with `ManagedPolicyArns` on the role avoids this restriction entirely.

---

## Step-by-Step Implementation

### Phase 1 -- Inspect Lambda Source Code

Before writing any CloudFormation, inspect the provided Lambda code to understand the handler name, runtime requirements, and environment variable dependencies.

```bash
cat /root/index.py
```

> **Screenshot Placeholder**
> `[SCREENSHOT-01: Output of cat /root/index.py showing full Lambda source code]`

**Key findings from inspection:**

* Handler name: `lambda_handler` --> CloudFormation `Handler` value: `index.lambda_handler`
* Runtime: Python 3.x (compatible with `python3.12`)
* Environment variables required: `high_priority_queue`, `low_priority_queue` (SQS queue URLs)
* Processing logic: polls High queue first; falls back to Low queue only when High is empty
* Uses `sqs.receive_message()` directly -- NOT event-driven; no `EventSourceMapping` needed
* `WaitTimeSeconds=3` -- Lambda timeout must exceed 3 seconds + execution overhead

---

### Phase 2 -- Author the CloudFormation Template

Write the complete CloudFormation template to `/root/nautilus-priority-stack.yml`:

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

  HighPriorityQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref NautilusHighPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt NautilusHighPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref NautilusPriorityQueuesTopic

  LowPriorityQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref NautilusLowPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt NautilusLowPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref NautilusPriorityQueuesTopic

  LambdaSQSSNSManagedPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: LambdaSQSSNSPolicy
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Action:
              - sqs:ReceiveMessage
              - sqs:DeleteMessage
              - sqs:GetQueueAttributes
              - sqs:ChangeMessageVisibility
            Resource:
              - !GetAtt NautilusHighPriorityQueue.Arn
              - !GetAtt NautilusLowPriorityQueue.Arn
          - Effect: Allow
            Action:
              - sns:Publish
              - sns:ListTopics
            Resource: !Ref NautilusPriorityQueuesTopic
          - Effect: Allow
            Action:
              - logs:CreateLogGroup
              - logs:CreateLogStream
              - logs:PutLogEvents
            Resource: arn:aws:logs:*:*:*

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
        - !Ref LambdaSQSSNSManagedPolicy

  NautilusPrioritiesQueueFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: nautilus-priorities-queue-function
      Runtime: python3.12
      Role: !GetAtt LambdaExecutionRole.Arn
      Handler: index.lambda_handler
      Timeout: 60
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

Outputs:
  HighPriorityQueueURL:
    Value: !Ref NautilusHighPriorityQueue
  LowPriorityQueueURL:
    Value: !Ref NautilusLowPriorityQueue
  SNSTopicArn:
    Value: !Ref NautilusPriorityQueuesTopic
  LambdaFunctionName:
    Value: !Ref NautilusPrioritiesQueueFunction
EOF
```

Verify the template was written correctly:

```bash
# Confirm file exists and has content
ls -lh /root/nautilus-priority-stack.yml

# Confirm environment variables are present
grep -A4 "Environment:" /root/nautilus-priority-stack.yml

# Confirm correct handler
grep "Handler:" /root/nautilus-priority-stack.yml

# Confirm no EventSourceMapping exists (must return empty)
grep "EventSourceMapping" /root/nautilus-priority-stack.yml
```

> **Screenshot Placeholder**
> `[SCREENSHOT-02: Output of ls -lh and grep verification commands confirming template contents]`

---

### Phase 3 -- Validate the Template

> **Note:** Python's `yaml.safe_load` does NOT understand CloudFormation intrinsic function tags like `!Ref` and `!GetAtt`. The resulting `ConstructorError` is expected and harmless. Use AWS CloudFormation validation exclusively.

```bash
aws cloudformation validate-template \
  --template-body file:///root/nautilus-priority-stack.yml \
  --region us-east-1
```

**Expected response:**

```json
{
    "Parameters": [],
    "Description": "Nautilus Priority Queuing with SQS, SNS, Lambda, and IAM",
    "Capabilities": ["CAPABILITY_NAMED_IAM"],
    "CapabilitiesReason": "The following resource(s) require capabilities: [AWS::IAM::ManagedPolicy]"
}
```

> **Screenshot Placeholder**
> `[SCREENSHOT-03: Successful AWS CloudFormation validate-template response showing CAPABILITY_NAMED_IAM]`

---

### Phase 4 -- Deploy the Stack

```bash
aws cloudformation create-stack \
  --stack-name nautilus-priority-stack \
  --template-body file:///root/nautilus-priority-stack.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

Wait for stack creation to complete:

```bash
aws cloudformation wait stack-create-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1 && echo "CREATE_COMPLETE"
```

Confirm final status:

```bash
aws cloudformation describe-stacks \
  --stack-name nautilus-priority-stack \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

> **Screenshot Placeholder**
> `[SCREENSHOT-04: Terminal showing CREATE_COMPLETE confirmation after stack deployment]`

**If the stack rolls back**, immediately retrieve the failure reason:

```bash
aws cloudformation describe-stack-events \
  --stack-name nautilus-priority-stack \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

---

### Phase 5 -- Verify All Resources

```bash
# Verify SQS Queues
aws sqs list-queues --queue-name-prefix nautilus --region us-east-1

# Verify SNS Topic
aws sns list-topics --region us-east-1 \
  --query "Topics[?contains(TopicArn,'nautilus-Priority-Queues-Topic')]"

# Verify Lambda function, handler, runtime, and environment variables
aws lambda get-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  --query '[FunctionName,Handler,Runtime,Environment]'

# Verify IAM Role
aws iam get-role --role-name lambda_execution_role \
  --query 'Role.[RoleName,Arn]'
```

> **Screenshot Placeholder**
> `[SCREENSHOT-05: All resource verification commands showing expected names, ARNs, and env vars]`

---

### Phase 6 -- Test Priority Queuing Behavior

#### Step 6.1 -- Publish Messages to SNS

```bash
topicarn=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn,'nautilus-Priority-Queues-Topic')].TopicArn" \
  --output text --region us-east-1)
echo "Topic ARN: $topicarn"

# Publish High Priority messages
aws sns publish --topic-arn $topicarn \
  --message 'High Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}' \
  --region us-east-1

aws sns publish --topic-arn $topicarn \
  --message 'High Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}' \
  --region us-east-1

# Publish Low Priority messages
aws sns publish --topic-arn $topicarn \
  --message 'Low Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}' \
  --region us-east-1

aws sns publish --topic-arn $topicarn \
  --message 'Low Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}' \
  --region us-east-1
```

> **Screenshot Placeholder**
> `[SCREENSHOT-06: SNS publish commands returning four unique MessageIds confirming delivery]`

#### Step 6.2 -- Invoke Lambda and Observe Priority Order

> **Important:** This environment uses **AWS CLI v1**. The flags `--payload` and `--cli-binary-format` are CLI v2 only and must NOT be used. The output file is a positional argument placed after all named flags.

```bash
# Invocation 1 -- expect High Priority message 1
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out1.json
cat /tmp/out1.json

# Invocation 2 -- expect High Priority message 2
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out2.json
cat /tmp/out2.json

# Invocation 3 -- High queue now empty, expect Low Priority message 1
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out3.json
cat /tmp/out3.json

# Invocation 4 -- expect Low Priority message 2
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --region us-east-1 \
  /tmp/out4.json
cat /tmp/out4.json
```

> **Screenshot Placeholder**
> `[SCREENSHOT-07: Invocations 1 and 2 output showing "High Priority message 1" and "High Priority message 2" deleted]`

> **Screenshot Placeholder**
> `[SCREENSHOT-08: Invocations 3 and 4 output showing "Low Priority message 1" and "Low Priority message 2" deleted, confirming fallback behavior]`

---

## Problems Encountered and Resolutions

### Problem 1 -- Stack Rollback: `iam:PutRolePolicy` Access Denied

**Symptom:**

```
ROLLBACK_COMPLETE
LambdaExecutionRole | Resource handler returned message:
"User: arn:aws:iam::691595780564:user/kk_labs_user_407114 is not authorized
to perform: iam:PutRolePolicy on resource: role lambda_execution_role"
```

> **Screenshot Placeholder**
> `[SCREENSHOT-09: describe-stack-events output showing iam:PutRolePolicy ACCESS_DENIED failure]`

**Root Cause:**

The initial template used an inline `Policies:` block inside `AWS::IAM::Role`. Inline policies are attached via the `iam:PutRolePolicy` API action, which the lab IAM user is not authorized to call.

**Resolution:**

Replaced the inline policy block with a separate `AWS::IAM::ManagedPolicy` resource and referenced it via `ManagedPolicyArns` on the role. Managed policy creation uses `iam:CreatePolicy` and `iam:AttachRolePolicy` -- both permitted for the lab user.

| Approach | IAM API Called | Permitted |
|---|---|---|
| Inline `Policies:` in role | `iam:PutRolePolicy` | No |
| `AWS::IAM::ManagedPolicy` + `ManagedPolicyArns` | `iam:CreatePolicy` + `iam:AttachRolePolicy` | Yes |

**Fix applied:**

```yaml
# REMOVED (caused AccessDenied):
LambdaExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    Policies:
      - PolicyName: LambdaSQSSNSPolicy
        PolicyDocument: ...

# REPLACED WITH:
LambdaSQSSNSManagedPolicy:
  Type: AWS::IAM::ManagedPolicy
  Properties:
    ManagedPolicyName: LambdaSQSSNSPolicy
    PolicyDocument: ...

LambdaExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    ManagedPolicyArns:
      - !Ref LambdaSQSSNSManagedPolicy
```

---

### Problem 2 -- AWS CLI v1 Incompatibility: `Unknown options: --cli-binary-format`

**Symptom:**

```
Unknown options: --cli-binary-format, /tmp/out1.json
```

**Root Cause:**

The initial Lambda invoke commands used `--cli-binary-format raw-in-base64-out` and `--payload '{}'` with the output file as a positional argument mixed with named flags -- all of which are AWS CLI v2 features. The client host runs AWS CLI v1.

**Resolution:**

Removed all CLI v2-specific flags. In CLI v1, the output file is a required positional argument placed at the end of the command after all named flags. The `--payload` flag is also unnecessary since the Lambda handler ignores the event input.

| CLI v2 Syntax (broken) | CLI v1 Syntax (correct) |
|---|---|
| `aws lambda invoke --payload '{}' --cli-binary-format raw-in-base64-out /tmp/out.json` | `aws lambda invoke --function-name ... --region us-east-1 /tmp/out.json` |

---

### Problem 3 -- Lambda Timeout on Empty Queue

**Symptom:**

```json
{
  "StatusCode": 200,
  "FunctionError": "Unhandled",
  "ExecutedVersion": "$LATEST"
}
{"errorType":"Sandbox.Timedout","errorMessage":"Task timed out after 3.00 seconds"}
```

**Root Cause:**

The Lambda `Timeout` in the template was set to 3 seconds. The `poll_messages()` function uses `WaitTimeSeconds=3` (long polling). When both queues are empty, the function blocks for exactly 3 seconds waiting for messages -- which equals the Lambda execution timeout, causing a race condition that results in a timeout error.

**Resolution:**

Updated Lambda timeout to 10 seconds post-deployment so the 3-second SQS long poll completes before the Lambda timeout fires:

```bash
aws lambda update-function-configuration \
  --function-name nautilus-priorities-queue-function \
  --timeout 10 \
  --region us-east-1
```

**Best practice:** Lambda timeout should always be at least `WaitTimeSeconds + 5` seconds of buffer. The CloudFormation template was already authored with `Timeout: 60` -- the issue arose only because invocation 4 was attempted when the 4th message (Low Priority message 2) was still in flight after the previous timeout interrupted before it could be consumed. Invocation 5 (after the timeout fix) successfully consumed it.

> **Screenshot Placeholder**
> `[SCREENSHOT-10: Invocation 5 output showing "Low Priority message 2" deleted with StatusCode 200 and no error]`

---

## Test Results

| Invocation | Queue Polled | Message Consumed | Priority Attribute | Status |
|---|---|---|---|---|
| 1 | High | `High Priority message 1` | `high` | 200 OK |
| 2 | High | `High Priority message 2` | `high` | 200 OK |
| 3 | High (empty) then Low | `Low Priority message 1` | `low` | 200 OK |
| 4 | Timeout (both queues temporarily empty) | -- | -- | Timeout |
| 5 (after fix) | High (empty) then Low | `Low Priority message 2` | `low` | 200 OK |

**Conclusion:** The SNS filter policies correctly routed high-priority messages exclusively to the High queue and low-priority messages exclusively to the Low queue. The Lambda function enforced priority-first consumption on every invocation.

---

## Best Practices

### Infrastructure as Code

* **Always inspect existing code before writing IaC.** Reading `index.py` before authoring the template revealed the environment variable dependency and the non-event-driven pattern -- both of which would have caused silent failures if assumed incorrectly.
* **Never assume IAM permissions in a scoped lab environment.** Always check what actions the deploying user is authorized to perform before choosing between inline policies vs. managed policies.
* **Parameterize reusable values.** Queue names, function names, and topic names should ideally be CloudFormation Parameters with defaults to allow reuse across environments.
* **Always include `Outputs`.** Exporting resource ARNs and URLs as CloudFormation Outputs makes cross-stack references and testing automation significantly easier.

### Lambda

* **Lambda `Timeout` must always exceed `WaitTimeSeconds` by a meaningful buffer** (minimum 5 seconds). If `WaitTimeSeconds = 3`, set `Timeout` to at least 10.
* **Validate environment variable dependencies at design time.** If the Lambda reads from `os.environ`, ensure the CloudFormation `Environment.Variables` block injects those values via `!Ref` or `!GetAtt`.
* **Do not add `EventSourceMapping` when Lambda polls SQS directly.** Event-driven (push) and polling (pull) are mutually exclusive patterns. Mixing them causes duplicate processing and unexpected behavior.

### SNS + SQS Fan-Out

* **Always create `AWS::SQS::QueuePolicy` alongside SNS subscriptions.** Without explicit `sqs:SendMessage` permission granted to the SNS service principal, SNS delivery to SQS silently fails.
* **Use `FilterPolicy` for routing, not separate topics.** A single SNS topic with per-subscription filter policies is more operationally efficient than maintaining multiple topics.
* **Always use `ArnEquals` condition on queue policies** to prevent unauthorized SNS topics from writing to your queues.

### AWS CLI

* **Detect CLI version before scripting.** Run `aws --version` to confirm v1 vs. v2. Flags like `--cli-binary-format`, `--no-cli-pager`, and query syntax differ between versions.
* **Use `aws cloudformation validate-template`** -- not Python's `yaml.safe_load` -- to validate CloudFormation templates. Python YAML parsers do not understand CloudFormation intrinsic function tags (`!Ref`, `!GetAtt`, `!Sub`).
* **Use `aws cloudformation wait stack-create-complete`** instead of polling `describe-stacks` in a loop. The `wait` command handles retries and exits cleanly on terminal states.

---

## Lessons Learned

### 1. Always Read the Code Before Writing the Template

The single most impactful step was inspecting `/root/index.py` before writing a single line of CloudFormation. This revealed:

* The Lambda uses `os.environ` -- environment variables are mandatory, not optional
* The Lambda polls SQS directly -- `EventSourceMapping` must NOT be added
* The handler name is `lambda_handler` inside a file named `index.py` -- `Handler: index.lambda_handler`

Skipping this step would have produced a template that deploys successfully but silently fails at runtime.

### 2. Scoped IAM Users Require Managed Policies

In shared or sandboxed AWS environments, `iam:PutRolePolicy` (inline policies) is commonly restricted while `iam:CreatePolicy` and `iam:AttachRolePolicy` (managed policies) remain available. Always default to `AWS::IAM::ManagedPolicy` in environments where full IAM privileges are not guaranteed.

### 3. Lambda Timeout Must Be Greater Than SQS WaitTimeSeconds

`WaitTimeSeconds` defines how long `receive_message` blocks waiting for messages. If Lambda's `Timeout` equals or is less than `WaitTimeSeconds`, empty-queue invocations will always time out. The rule is:

```
Lambda Timeout > WaitTimeSeconds + expected processing time + buffer
```

### 4. SNS Wraps the Message Body in a JSON Envelope

When SNS delivers to SQS, the SQS `Body` field is not the raw message -- it is a JSON-encoded SNS notification envelope containing the original `Message`, `MessageId`, `TopicArn`, `Timestamp`, `MessageAttributes`, and more. Any Lambda code that processes message content must parse this envelope to extract the original payload.

### 5. AWS CLI Version Compatibility Is a Runtime Concern

Flag incompatibilities between CLI v1 and v2 produce misleading `Unknown options` errors. Always verify the CLI version on the target host and test commands interactively before embedding them in automation scripts.

### 6. CloudFormation Stack Rollback Is a Diagnostic Tool, Not a Failure

A `ROLLBACK_COMPLETE` state is CloudFormation protecting you from a partial deployment. The correct response is always to run `describe-stack-events` with a `CREATE_FAILED` filter, identify the root cause from the `ResourceStatusReason`, delete the rolled-back stack, fix the template, and redeploy. Never attempt to update a `ROLLBACK_COMPLETE` stack.

---

## Resource Cleanup

To tear down all resources created by this lab and avoid ongoing AWS charges:

```bash
aws cloudformation delete-stack \
  --stack-name nautilus-priority-stack \
  --region us-east-1

aws cloudformation wait stack-delete-complete \
  --stack-name nautilus-priority-stack \
  --region us-east-1

echo "Stack and all resources deleted successfully"
```

---

<img width="1030" height="700" alt="image" src="https://github.com/user-attachments/assets/b89e922f-b0f1-4929-a5e8-c0e6aaf70f57" />
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


