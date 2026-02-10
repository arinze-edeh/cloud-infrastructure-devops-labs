# Enable EC2 Termination Protection

## Lab Overview
This lab demonstrates how to enable **termination protection** on an existing Amazon EC2 instance using the AWS Management Console.  
Termination protection prevents accidental deletion of critical compute resources.

---

## Environment Details

| Item | Value |
|-----|------|
| Cloud Provider | AWS |
| Service | EC2 |
| Region | us-east-1 |
| Instance Name | devops-ec2 |
| Action | Enable Termination Protection |

---

## Objective

- Locate an existing EC2 instance
- Enable termination protection
- Verify protection status

---

## High-Level Logic

- LOGIN to AWS Console
- SELECT correct AWS region

- NAVIGATE to EC2 instances
- IDENTIFY target instance

- IF termination protection is disabled:
  -  ENABLE termination protection

- VERIFY configuration
- CONFIRM success

## 🛠️ Step-by-Step Implementation

## Step 1: AWS Console Login

📸 Screenshot:
<img width="1735" height="942" alt="image" src="https://github.com/user-attachments/assets/392848b2-e3f9-4ce2-88ed-86b20ebb5a0f" />

## Step 2: Select Region (us-east-1)
📸 Screenshot:
<img width="1764" height="948" alt="image" src="https://github.com/user-attachments/assets/851eb819-a983-424c-9e5b-9cfe03b7f445" />

## Step 3: Open EC2 Dashboard
📸 Screenshot:
<img width="1763" height="946" alt="image" src="https://github.com/user-attachments/assets/2e1bd221-5380-4a9f-85b3-47409ae0f184" />

## Step 4: Locate EC2 Instance
📸 Screenshot:
<img width="1774" height="945" alt="image" src="https://github.com/user-attachments/assets/b975b96d-3dd5-4971-8f50-cef6eadf7876" />

## Step 5: Enable Termination Protection
📸 Screenshot:
<img width="1775" height="943" alt="image" src="https://github.com/user-attachments/assets/9d09b366-7c5e-4d08-bde2-cb69bab88cce" />

## Step 6: Verify Protection Status
📸 Screenshot:
<img width="1791" height="946" alt="image" src="https://github.com/user-attachments/assets/bdb09f3a-edf7-4be6-88d4-6a6dbf2dd8db" />

## ✅ Outcome
- EC2 termination protection enabled
- Accidental termination prevented
- Task completed successfully

## Key AWS Concepts Demonstrated
- EC2 instance management

- AWS region awareness

- Instance safety controls

- Infrastructure protection best practices

## Tags
`aws` `ec2` `compute` `cloud-security` `infrastructure`









