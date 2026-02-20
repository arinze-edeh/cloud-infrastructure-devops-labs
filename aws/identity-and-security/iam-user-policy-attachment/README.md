# AWS IAM User Policy Attachment (AWS CLI)

## 📌 Project Overview

- This project demonstrates how to securely attach an existing AWS IAM policy to an existing IAM user using the AWS CLI.
- The task validates identity, locates the policy, attaches it to the user, and verifies successful attachment — following IAM best practices.

- All actions were executed exclusively in the `us-east-1` region.

## 🎯 Objective

- Verify active AWS identity

- Locate an existing IAM policy

- Attach the policy to an IAM user

- Confirm successful policy attachment

## 🛠️ Tools & Technologies

`AWS IAM`

`AWS CLI`

`AWS STS`

## 📂 Resources Used

| Resource Type | Name |
|--------------|------|
| IAM User     | `iamuser_rose` |
| IAM Policy   | `iampolicy_rose` |
| AWS Region   | `us-east-1` |

##  Implementation Steps

### Step 1: Verify AWS Identity

- Ensured the correct AWS account and IAM identity were active before making changes.

- `aws sts get-caller-identity`

📸 Screenshot:
<img width="1027" height="563" alt="image" src="https://github.com/user-attachments/assets/db3e9d50-d012-4996-b5a0-f138450035dd" />


### Step 2: Retrieve IAM Policy ARN

- Queried IAM to confirm the policy exists and retrieved its ARN.

- `aws iam list-policies \`
  -  `--scope Local \`
  -  `--query "Policies[?PolicyName=='iampolicy_rose'].Arn" \`
  -  `--output text`

📸 Screenshot:
<img width="1031" height="724" alt="image" src="https://github.com/user-attachments/assets/31eb591b-2e09-40b2-a741-0175147ba6e9" />


### Step 3: Attach Policy to IAM User

- Attached the IAM policy to the target user.

- `aws iam attach-user-policy \`
  -  `--user-name iamuser_rose \`
  -  `--policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/iampolicy_rose`

📸 Screenshot:
<img width="1035" height="711" alt="image" src="https://github.com/user-attachments/assets/de834476-2d35-48c5-b547-2a30dbfe5d16" />


### Step 4: Verify Policy Attachment

- Confirmed the policy was successfully attached to the IAM user.

- `aws iam list-attached-user-policies \`
  -  `--user-name iamuser_rose`

📸 Screenshot:
<img width="1035" height="633" alt="image" src="https://github.com/user-attachments/assets/3a8d52ae-1a06-4ad8-bd00-86af7127f838" />

## ✅ Result

- IAM user identity verified

- Policy successfully located

- Policy attached to user

- Attachment confirmed via AWS CLI

- The IAM user now inherits all permissions defined in the attached policy.

## 🔐 Security & Best Practices

- Least-privilege access model followed

- No new IAM users or policies created

- Changes verified immediately after execution

- Region-restricted execution
