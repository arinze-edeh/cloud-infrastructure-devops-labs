# AWS EC2 Volume Attachment (EBS)

## 📌 Lab Overview

- The Nautilus DevOps team is managing AWS infrastructure using
incremental migration tasks. As part of storage management,
an existing EBS volume must be attached to an already running
EC2 instance using AWS tooling in the us-east-1 region.

- This lab validates correct identification, attachment, and
verification of block storage resources.

## 🎯 Objectives

- Identify an existing EC2 instance: `nautilus-ec2`

- Identify an existing EBS volume: `nautilus-volume`

- Attach the EBS volume to the EC2 instance

- Set the correct device name during attachment (/dev/sdb)

- Verify successful attachment status

## 🧠 High-Level Logic
- CONFIGURE AWS CLI credentials
- SET region to us-east-1

- LOCATE EC2 instance by tag "Name=nautilus-ec2"
- CONFIRM instance exists AND retrieve InstanceID

- LOCATE EBS volume by tag "Name=nautilus-volume"
- CONFIRM volume is available AND retrieve VolumeID

- ATTACH volume to instance
- SET device name to /dev/sdb

- VERIFY volume state is "attached"
- CONFIRM attachment is successful via CLI
  
## 🛠️ Implementation Steps

## Step 1: Verify Region

- Ensure the correct region is set.
- `aws configure get region`

📸 screenshot:
<img width="1031" height="686" alt="image" src="https://github.com/user-attachments/assets/aa34b752-7134-4b73-90af-19b23af74e75" />

## Step 2: Identify EC2 Instance

- Retrieve the instance ID for the target EC2 instance.
- `INSTANCE_ID=$(aws ec2 describe-instances \`
  -  `--filters "Name=tag:Name,Values=nautilus-ec2" \`
  -  `--query "Reservations[].Instances[].InstanceId" \`
  -  `--output text)`
-  `echo $INSTANCE_ID`

📸 screenshots:
<img width="1029" height="678" alt="image" src="https://github.com/user-attachments/assets/093ab8b8-b14e-4273-9852-42afc5f93ce0" />

## Step 3: Identify EBS Volume

- Retrieve the volume ID for the existing EBS volume.

- `VOLUME_ID=$(aws ec2 describe-volumes \`
  -  `--filters "Name=tag:Name,Values=nautilus-volume" \`
  -  `--query "Volumes[].VolumeId" \`
  -  `--output text)`
  - `echo $VOLUME_ID`

📸 screenshot:
<img width="1025" height="613" alt="image" src="https://github.com/user-attachments/assets/2b4385b3-6293-4358-9083-02849ae476a8" />

## Step 4:Attach Volume to EC2 Instance

- Attach the volume to the instance using the required device name.

- `aws ec2 attach-volume \`
  -  `--volume-id $VOLUME_ID \`
  -  `--instance-id $INSTANCE_ID \`
  -  `--device /dev/sdb`

📸 screenshot:
<img width="1033" height="756" alt="image" src="https://github.com/user-attachments/assets/fad3c318-9cd6-4038-99a4-d58bec0b3c12" />

## Step 5: Verify Volume Attachment

- Confirm the volume is attached.

- aws ec2 attach-volume \
  -  --volume-id <VOLUME_ID> \
  -  --instance-id <INSTANCE_ID> \
  -  --device /dev/sdb


📸 screenshot:


## ✅ Final Outcome

- Existing EBS volume successfully attached to EC2 instance

- Device name correctly set to `/dev/sdb`

- Volume state confirmed as `attached`

- Task completed within the `us-east-1` region

## 🏷️ Tags
`aws` `ec2` `ebs` `storage` `cloud` `devops` `infrastructure`






<img width="1024" height="836" alt="image" src="https://github.com/user-attachments/assets/8e918200-00e1-41df-8b11-620764529996" />
<img width="1027" height="813" alt="image" src="https://github.com/user-attachments/assets/c7cf1337-8ce1-426a-b77b-47bac3402406" />
<img width="1038" height="861" alt="image" src="https://github.com/user-attachments/assets/7596bc69-0f0c-4039-b083-0fe2ee761fcd" />

