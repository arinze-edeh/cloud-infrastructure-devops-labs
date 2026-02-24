# AWS S3 Bucket Data Migration & Sync

## Project Overview

As part of a data migration initiative, this project demonstrates how to:
- Create a **new private S3 bucket**
- Migrate all data from an existing S3 bucket
- Verify **data consistency and integrity**
- Perform all operations using **AWS CLI**

This task is executed entirely in the **us-east-1** region.

---

## Architecture Diagram (Logical)

SOURCE S3 BUCKET
    |
    |  aws s3 sync
    v
DESTINATION S3 BUCKET (PRIVATE)

---

## Prerequisites

- AWS CLI installed
- IAM user with:
  - S3 full access
- Configured AWS credentials
- Correct AWS region set

---

## Step 1: Verify AWS CLI Authentication

- `aws sts get-caller-identity`

📸 Screenshot:
<img width="1036" height="469" alt="image" src="https://github.com/user-attachments/assets/2ef857c9-3fdb-4f53-8984-e59dfb9d55b8" />

## Step 2: Create the Destination Bucket
- `aws s3 mb s3://xfusion-sync-10728 --region us-east-1`
📸 Screenshot:
<img width="1032" height="673" alt="image" src="https://github.com/user-attachments/assets/dbc26f33-dad8-4508-ba1e-3fb8c9d4d34e" />

## Step 3: Create New Private S3 Bucket
aws s3 mb s3://xfusion-sync-10728 --region us-east-1

📸 Screenshot Placeholder:

screenshots/03-create-bucket.png

## Step 4: Verify Source Bucket Contents
aws s3 ls s3://xfusion-s3-16544

📸 Screenshot Placeholder:

screenshots/04-source-bucket-list.png

## Step 5: Sync Data from Source to Destination
aws s3 sync s3://xfusion-s3-16544 s3://xfusion-sync-10728

📸 Screenshot Placeholder:

screenshots/05-sync-operation.png

## Step 6: Validate Destination Bucket Data
aws s3 ls s3://xfusion-sync-10728 --recursive

📸 Screenshot Placeholder:

screenshots/06-destination-bucket-list.png

## Step 7: Verify Data Consistency (Optional Dry Run)
aws s3 sync s3://xfusion-s3-16544 s3://xfusion-sync-10728 --dryrun

📸 Screenshot Placeholder:

screenshots/07-dry-run-check.png

## Validation Checklist

- Destination bucket created successfully

- Bucket is private

- All objects copied successfully

- Object count matches source

- No sync errors reported

## Key AWS Services Used

- Amazon Web Services

- Amazon S3

- AWS CLI

- IAM

## Outcome

- Successful migration of all objects
-  No data loss or corruption
- Data consistency verified




<img width="1028" height="648" alt="image" src="https://github.com/user-attachments/assets/49a14bd2-4b55-48d0-be54-9e4fb0d8a44b" />
<img width="1026" height="828" alt="image" src="https://github.com/user-attachments/assets/99237cf2-97d8-4081-86d1-929c97df4280" />
<img width="1023" height="865" alt="image" src="https://github.com/user-attachments/assets/eca421ab-64eb-4be7-9643-ac847db4ed84" />
<img width="1032" height="499" alt="image" src="https://github.com/user-attachments/assets/c1122a7d-f7dd-4f4d-bc5b-a6055f2dd381" />
<img width="1000" height="857" alt="image" src="https://github.com/user-attachments/assets/104fbab7-db51-4bcd-8601-ebc27ac917fb" />
<img width="1025" height="335" alt="image" src="https://github.com/user-attachments/assets/fae5892a-d31c-4285-a7db-98263a3e8aef" />
