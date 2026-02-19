# IAM Read-Only EC2 Policy (AWS CLI)

PROJECT TYPE:
    `AWS Identity and Security`

SERVICE:
    `AWS IAM`

REGION:
   ` us-east-1`

POLICY NAME:
    `iampolicy_kirsty`

IMPLEMENTATION METHOD:
    `AWS CLI (Command Line Interface)`

---

## Project Overview

- This project demonstrates how to create a custom AWS IAM policy
using the AWS CLI.

- The policy grants **read-only access** to Amazon EC2 resources,
specifically allowing users to view:
  -  EC2 instances
  -  AMIs
  -  Snapshots

- No write, modify, or delete permissions are included.

- This lab emphasizes:
  -  IAM fundamentals
  -  Least-privilege access
  -  Infrastructure administration via CLI

---

## Problem Statement

- The DevOps team requires a custom IAM policy that:
  -  Allows read-only visibility into EC2 resources
  -  Is created programmatically using AWS CLI
  -  Exists only in the us-east-1 region

---

## Objectives

- Authenticate to AWS using temporary credentials
- Define an IAM policy in JSON format
- Create the policy using AWS CLI
- Verify successful policy creation
- Confirm policy availability in IAM

---

## Environment Details

AWS ACCOUNT:
    `Lab-provided AWS account`

REGION:
    `us-east-1`

AUTHENTICATION METHOD:
    `Temporary AWS console credentials`

TOOLS USED:
    - AWS CLI
    - Linux shell

---

## Step-by-Step Implementation

---

### Step 1: Retrieve AWS Credentials

ACTION:
    Run command to display temporary AWS credentials

COMMAND:
    `showcreds`

OUTPUT:
    - AWS Console URL
    - AWS Username
    - AWS Password
    - Session expiration time

SCREENSHOT: `showcreds output`
<img width="1029" height="685" alt="551840440-7a75c81b-dd0c-438b-97b0-7b143515fd8f" src="https://github.com/user-attachments/assets/7606d269-008c-4f0a-9816-05412a6b5988" />

---

### Step 2: Create IAM Policy JSON File

ACTION:
    Create policy.json file using shell redirection

COMMAND:
-    `cat <<EOF > policy.json`

POLICY CONTENT:
 `{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
               "ec2:DescribeInstances",
                "ec2:DescribeImages",
                "ec2:DescribeSnapshots"
            ],
            "Resource": "*"
        }
    ]
 }

 EOF

PURPOSE:
    Define read-only EC2 permissions

SCREENSHOT: `policy.json file contents`
<img width="1034" height="650" alt="image" src="https://github.com/user-attachments/assets/5120b923-cb2b-4a57-85ee-4945fb56c245" />

---

### Step 3: Create IAM Policy Using AWS CLI

ACTION:
    Use AWS CLI to create IAM policy

COMMAND:
    aws iam create-policy \
        --policy-name iampolicy_kirsty \
        --policy-document file://policy.json

EXPECTED OUTPUT:
    - PolicyName
    - PolicyId
    - Policy ARN
    - CreateDate

SCREENSHOT: `create-policy command output`
<img width="1032" height="770" alt="image" src="https://github.com/user-attachments/assets/8cb66d29-4ae3-46ab-81fb-b02f271fe667" />

---

### Step 4: Verify Policy Creation

ACTION:
    List local IAM policies and filter by policy name

COMMAND:
  -  `aws iam list-policies \`
      -  `--scope Local \`
      -  `--query 'Policies[?PolicyName==`iampolicy_kirsty`].Arn' \`
      -  `--output text`

EXPECTED RESULT:
    `arn:aws:iam::<account-id>:policy/iampolicy_kirsty`

SCREENSHOT: `policy verification output`
<img width="1033" height="858" alt="image" src="https://github.com/user-attachments/assets/092cc3e3-20f5-49e3-be83-965eaea2cf0e" />

---

## Validation Checklist

- Policy JSON file created successfully
- IAM policy created using AWS CLI
- Correct policy name applied
- Read-only EC2 permissions confirmed
- Policy visible in IAM policy list
- Region set to `us-east-1`

---

## Outcome

- The IAM policy **iampolicy_kirsty** was successfully created
using the AWS CLI.

- The policy grants secure, read-only access to EC2 resources,
supporting operational visibility while enforcing least privilege.

- This lab demonstrates practical IAM policy management
in a real-world DevOps workflow.










