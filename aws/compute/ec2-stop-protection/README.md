# AWS EC2 – Enable Stop Protection (Management Console)

## Lab Overview
As part of an infrastructure migration, this lab demonstrates how to enable **Stop Protection** for an existing Amazon EC2 instance using the **AWS Management Console**.

Stop Protection prevents an EC2 instance from being accidentally stopped through the console, CLI, or API.

---

## Cloud Provider
- **Platform:** Amazon Web Services (AWS)
- **Service:** EC2
- **Region:** us-east-1

---

## Requirements
- Existing EC2 instance named `datacenter-ec2`
- AWS Console access
- Permissions to modify EC2 instance settings

---

## Task Objective
Enable **Stop Protection** for the EC2 instance:
- **Instance Name:** datacenter-ec2
- **Region:** us-east-1

---

## Step-by-Step Implementation (AWS Console)

### Step 1: Log in to AWS Console
- Open the AWS Console URL
- Sign in with the provided credentials
- Ensure the region is set to **us-east-1**

📸 **Screenshot:**  
`AWS Console Dashboard showing region set to us-east-1`

<img width="1750" height="946" alt="image" src="https://github.com/user-attachments/assets/4b1604ea-0fce-4ccf-a45a-eee6fefbfa9e" />

---

### Step 2: Navigate to EC2 Service
- Open **Services** → **EC2**
- Click **Instances** from the left navigation panel

📸 **Screenshot:**  
`EC2 service dashboard`

<img width="1831" height="946" alt="image" src="https://github.com/user-attachments/assets/ac2f1de9-3452-40dc-8e01-f03fb34de002" />

---

### Step 3: Select the EC2 Instance
- Locate the instance named **datacenter-ec2**
- Select the instance by checking the box

📸 **Screenshot:**  
`EC2 instances list showing datacenter-ec2 selected`

<img width="1780" height="946" alt="image" src="https://github.com/user-attachments/assets/22d1cd0d-eaf3-4634-9c60-2cd99bc09d7f" />
<img width="1776" height="948" alt="image" src="https://github.com/user-attachments/assets/1d1a6ede-3107-4a64-928a-5689d64c279d" />

---

### Step 4: Enable Stop Protection
- Click **Actions**
- Navigate to **Instance settings**
- Select **Change stop protection**
- Check **Enable stop protection**
- Click **Save**

📸 **Screenshot:**  
`Change stop protection dialog with enable option selected`

<img width="1795" height="948" alt="image" src="https://github.com/user-attachments/assets/efbec68f-7642-4606-ae5a-bc2cf83860da" />
<img width="1794" height="947" alt="image" src="https://github.com/user-attachments/assets/5e17501a-8a22-40ef-94ac-814036bfef41" />

---

### Step 5: Verify Configuration
- Select the instance
- Open the **Details** tab
- Confirm **Stop protection: Enabled**

📸 **Screenshot:**  
`Instance details showing stop protection enabled`

<img width="1798" height="942" alt="image" src="https://github.com/user-attachments/assets/b31f144f-ec53-46fd-8183-a731e3cb71a6" />

---

## Final Outcome
- Stop Protection successfully enabled
- Instance cannot be stopped accidentally
- Configuration validated via EC2 instance details

---

## Key Takeaways
- Stop Protection adds a safety layer for critical workloads
- Configuration can be managed directly from the AWS Console
- Useful for production or sensitive environments
