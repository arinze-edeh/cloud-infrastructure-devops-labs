# AWS IAM Group Management Lab

## 📌 Project Overview
- This project demonstrates the use of the **AWS Command Line Interface (CLI)** to manage **Identity and Access Management (IAM)** resources in AWS.  
- The primary objective of this lab is to create and verify an IAM group using secure, programmatic access rather than the AWS Management Console.

- The lab was executed in the **us-east-1 (N. Virginia)** region using temporary AWS credentials.

---

## 🎯 Objectives
- Authenticate into AWS using temporary credentials
- Create an IAM group using AWS CLI
- Verify the IAM group creation
- Understand basic IAM group management workflows

---

## 🛠️ Tools & Technologies
- **AWS CLI**
- **AWS IAM**
- **Linux Shell (Cloud-based lab environment)**
- **Region:** us-east-1

---

## 🔐 Authentication
Temporary AWS credentials were generated and used to access the AWS account via the CLI.

AWS Console URL: https://954973595150.signin.aws.amazon.com/console
Region: us-east-1
`⚠️ Note: Credentials shown in this lab are temporary and automatically expire.`

## 🚀 Implementation Steps

## Step 1: Verify AWS Credentials
- The showcreds command was used to confirm active AWS credentials and session validity.

`showcreds`

Screenshots: 
<img width="1028" height="665" alt="image" src="https://github.com/user-attachments/assets/36733923-687f-4a6e-8e18-6780d182edf9" />


- This confirmed:

  -  Active AWS account

  -  IAM user access

  -  Session expiration time

## Step 2: Create an IAM Group
An IAM group named iamgroup_ammar was created using the AWS CLI.

`aws iam create-group --group-name iamgroup_ammar`

Screenshots: 
<img width="1028" height="541" alt="image" src="https://github.com/user-attachments/assets/05e1715b-a278-4f71-b674-7e764e8041f2" />

## ✅ Result:

- IAM group successfully created

- Group ARN and ID returned by AWS

## Step 3: Verify IAM Group Creation
- The group was queried to confirm its existence and current membership.

`aws iam get-group --group-name iamgroup_ammar`

Screenshots: 

<img width="1031" height="626" alt="image" src="https://github.com/user-attachments/assets/57fa7f74-ce12-485d-8c05-1372bb10fad5" />

## ✅ Result:

- Group exists

- No users assigned yet

## 📂 Project Output
- Successfully created IAM group: iamgroup_ammar

- Group verified with zero users attached

- Demonstrated correct use of AWS CLI IAM commands

## 🔎 Key Learnings

- How to manage IAM resources using AWS CLI

- Importance of IAM groups in access control

- Verifying AWS resource creation programmatically

- Working with temporary AWS credentials securely








