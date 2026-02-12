# Attach Existing ENI to EC2 Instance

## 📌 Lab Overview
- As part of an incremental AWS migration, the Nautilus DevOps team
required an existing Elastic Network Interface (ENI) to be attached
to an already running EC2 instance.

- This lab demonstrates how to attach an ENI to an EC2 instance
using the AWS CLI and verify successful attachment.

---

## 🎯 Objectives
- Identify existing EC2 instance
- Identify existing ENI
- Attach ENI to the EC2 instance
- Verify ENI status is `attached`

---

## 🧠 High-Level Logic

- AUTHENTICATE to AWS
- SET region to us-east-1

- FETCH EC2 instance ID
- FETCH ENI ID

- ATTACH ENI to EC2 instance
- VERIFY ENI attachment status

## 🛠️ Implementation Steps

## Step 1: Authenticate AWS CLI
- aws configure

📸 screenshot:
<img width="1040" height="644" alt="image" src="https://github.com/user-attachments/assets/da70b1ed-7e8b-442e-8cae-779c76ebdc0e" />

## Step 2: Verify AWS Region
- aws configure get region

📸 screenshot:
<img width="1032" height="436" alt="image" src="https://github.com/user-attachments/assets/0f796f77-aa0f-46ab-b7bc-cd14170b0bd7" />

## Step 3: Retrieve EC2 Instance ID
- aws ec2 describe-instances \
  -  --filters "Name=tag:Name,Values=nautilus-ec2" \
  -  --query "Reservations[].Instances[].InstanceId" \
  -  --output text

📸 screenshot:
<img width="1039" height="545" alt="image" src="https://github.com/user-attachments/assets/c411cd87-4d10-4de1-abf3-3dca3d046baf" />

## Step 4: Retrieve ENI ID
- aws ec2 describe-network-interfaces \
  -  --filters "Name=tag:Name,Values=nautilus-eni" \
  -  --query "NetworkInterfaces[].NetworkInterfaceId" \
  -  --output text

📸 screenshot:
<img width="1042" height="622" alt="image" src="https://github.com/user-attachments/assets/8dddd358-87d6-4f2e-b318-caed828c8f67" />

## Step 5: Attach ENI to EC2
- aws ec2 attach-network-interface \
  --network-interface-id <eni-id> \
  --instance-id <instance-id> \
  --device-index 1

📸 screenshot:
<img width="1042" height="661" alt="image" src="https://github.com/user-attachments/assets/69392dba-668e-462a-988d-288f1cda754d" />

## Step 6: Verify ENI Attachment Status
aws ec2 describe-network-interfaces \
  --network-interface-ids <eni-id> \
  --query "NetworkInterfaces[].Status" \
  --output table

📸 screenshot:
<img width="1037" height="870" alt="image" src="https://github.com/user-attachments/assets/1a2c8142-8257-4b03-b496-d9ce358d359d" />
<img width="1039" height="868" alt="image" src="https://github.com/user-attachments/assets/3a3335b9-b1f1-4f83-9fc1-fb0a69a699b7" />

## ✅ Final Outcome
- Existing ENI successfully attached to EC2 instance

- ENI status verified as attached

- Task completed within us-east-1 region

## 🏷️ Tags
- aws ec2 eni networking devops









