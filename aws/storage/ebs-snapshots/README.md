# AWS EBS Snapshot Creation – DevOps Volume

## 📌 Lab Overview

The Nautilus DevOps team requires regular backups of critical AWS volumes.
As part of the initial backup strategy, a manual snapshot is created for an
existing EBS volume in the `us-east-1` region.

## 🎯 Objectives

- Identify existing EBS volume
- Create snapshot with required name and description
- Ensure snapshot creation completes successfully

## 🧠 High-Level Logic
- IF target volume exists in correct region:
  -  CREATE snapshot
  -  WAIT until snapshot status is completed
- ELSE:
  -  FAIL task

## 🛠️ Implementation Steps

## Step 1: Login to AWS Console
- LOGIN using provided AWS credentials
- SET region to us-east-1

screenshot: `aws-console-login`
<img width="1819" height="948" alt="image" src="https://github.com/user-attachments/assets/fe0f1630-0b5b-4fef-a438-8c3499ca9dc2" />

## Step 2: Locate EBS Volume
- NAVIGATE to EC2 Dashboard
- OPEN Volumes section
- SEARCH for devops-vol

screenshot: `ebs-volume-list`
<img width="1845" height="947" alt="image" src="https://github.com/user-attachments/assets/2950c000-0254-4bfa-9a54-cb3a32df03b9" />

## Step 3: Create Snapshot
- SELECT volume devops-vol
- CLICK Create Snapshot

- SET:
  -  Name = `devops-vol-ss`
  -  Description = `devops Snapshot`

screenshots: `create-snapshot-form`
<img width="1845" height="945" alt="image" src="https://github.com/user-attachments/assets/7436774a-0f71-4175-be4f-04c178bac0c3" />
<img width="1842" height="944" alt="image" src="https://github.com/user-attachments/assets/b003fca4-bf49-413c-af81-fb75a52b0e4f" />

## Step 4: Verify Snapshot Status
- NAVIGATE to Snapshots section
- CONFIRM snapshot status = completed

screenshot: `snapshot-completed-status`
<img width="1841" height="948" alt="image" src="https://github.com/user-attachments/assets/eeac0f39-5b70-4a8f-b265-7c167ec1a174" />

## ✅ Final Outcome

- Snapshot created successfully
- Snapshot name matches requirement
- Snapshot description applied correctly
- Snapshot status confirmed as completed

## 🏷️ Tags

`aws`
`ebs`
`snapshots`
`storage`
`backup`
`cloud-infrastructure`
`devops`
