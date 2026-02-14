# EC2 AMI Creation from Existing Instance

## 📌 Lab Overview
- The Nautilus DevOps team is migrating infrastructure to AWS using an
incremental approach. As part of this process, an Amazon Machine Image
(AMI) was created from an existing EC2 instance to enable repeatable
deployments, backups, and scaling operations.

---

## 🎯 Objectives
- Identify an existing EC2 instance
- Create an AMI from the instance
- Ensure AMI reaches `available` state
- Validate successful image creation

---

## 🧠 High-Level Logic

- LOGIN to AWS Console
- SELECT region us-east-1

- LOCATE EC2 instance "nautilus-ec2"

- INITIATE AMI creation:
  -  SET image name = "nautilus-ec2-ami"
  -  START image creation process

- WAIT until AMI status becomes "available"

- VERIFY AMI exists and is usable

## 🛠️ Implementation Steps

## Step 1: Login to AWS Console
- Open AWS Console URL

- Authenticate using provided credentials

- Ensure region is set to us-east-1

📸 screenshot:
<img width="1791" height="941" alt="image" src="https://github.com/user-attachments/assets/30502e7a-8fae-4566-93dc-bfeddb7d7a1c" />

## Step 2: Navigate to EC2 Service
- Open EC2 Dashboard

- Select Instances

- Locate instance named nautilus-ec2

📸 screenshot:
<img width="1785" height="937" alt="image" src="https://github.com/user-attachments/assets/cf69143b-126a-4e75-998d-2171c3803bab" />

## Step 3: Create AMI from EC2 Instance
- Select the instance nautilus-ec2

- Click Actions

- Choose Image and templates

- Click Create image

📸 screenshot:


## Step 4: Configure AMI Details
- AMI Name: nautilus-ec2-ami

- Leave other settings as default

- Click Create image

📸 screenshot:

## Step 5: Monitor AMI Creation Status
- Navigate to AMIs

- Locate nautilus-ec2-ami

- Wait until status changes to available

📸 screenshot:

## Step 6: Verify AMI Availability
- Confirm AMI state is available

- Ensure AMI is owned by current account

- Validate readiness for EC2 launches

📸 screenshot:

## ✅ Final Outcome
- AMI successfully created from existing EC2 instance

- AMI name set correctly as nautilus-ec2-ami

- Image reached available state

- AMI ready for reuse, scaling, and recovery operations

🏷️ Tags
`aws` `ec2` `ami` `snapshots` `cloud-migration` `devops` `infrastructure`




<img width="1793" height="944" alt="image" src="https://github.com/user-attachments/assets/1e8e3179-629c-4970-9e98-c6b755bf1a7e" />
<img width="1821" height="943" alt="image" src="https://github.com/user-attachments/assets/86838675-4cf1-4eb6-b4a5-9e1ad27cf5b4" />
<img width="1856" height="940" alt="image" src="https://github.com/user-attachments/assets/0b007ee7-b952-4cda-8860-d366cf4a3861" />
<img width="1861" height="945" alt="image" src="https://github.com/user-attachments/assets/8f892b1d-576d-4f36-b16a-1e6400649aaf" />

