# AWS S3 Data Protection – Bucket Versioning (CLI)

## Overview
This lab demonstrates how to enable Amazon S3 bucket versioning using the AWS CLI.
Versioning is a critical data-protection mechanism that allows recovery from
accidental deletions, overwrites, and data corruption in production environments.

The task reflects a real-world DevOps scenario where data durability and
recoverability are required for managed cloud storage resources.

---

## Business Context
- Data protection and recovery are fundamental requirements for modern cloud systems.
- The DevOps team was tasked with improving resilience for an existing S3 bucket
  by enabling object versioning.
- This change ensures historical object versions can be restored when needed.

---

## Objectives
- Verify the target S3 bucket exists
- Check current versioning state
- Enable S3 bucket versioning using AWS CLI
- Confirm versioning is successfully enabled

---

## Tools & Services Used
- AWS S3
- AWS CLI
- Linux Shell
- IAM (preconfigured lab credentials)

---

## Environment Details
- Cloud Provider: AWS
- Service: Amazon S3
- Region: us-east-1
- Bucket Name: nautilus-s3-9397
- Access Method: AWS CLI

---

## Step 1: Confirm Active AWS Region

  verify aws region is set
  ensure region equals us-east-1

Command:
  aws configure get region

Expected Result:
  us-east-1

Screenshot:
 <img width="1046" height="864" alt="image" src="https://github.com/user-attachments/assets/9b259c85-fe37-4c5f-8228-691da9ae2b4d" />

---

## Step 2: Verify Target S3 Bucket Exists

  list all s3 buckets
  confirm target bucket is present

Command:
  aws s3 ls

Expected Result:
  nautilus-s3-9397 is listed

Screenshot:
 <img width="1035" height="869" alt="image" src="https://github.com/user-attachments/assets/a9a975f2-dd3d-4918-8e64-7d56b26d567f" />

---

## Step 3: Check Current Versioning Status

  query bucket versioning configuration
  determine if versioning is disabled or suspended

Command:
  aws s3api get-bucket-versioning \
    --bucket nautilus-s3-9397

Expected Result (before change):
  empty output

Screenshot:
  <img width="1031" height="850" alt="image" src="https://github.com/user-attachments/assets/1626b4ac-67bd-4f62-90f6-1ae061c2cf5f" />

---

## Step 4: Enable S3 Bucket Versioning

  apply versioning configuration
  set status to Enabled

Command:
  aws s3api put-bucket-versioning \
    --bucket nautilus-s3-9397 \
    --versioning-configuration Status=Enabled

Expected Result:
  command executes successfully with no output

Screenshot:
<img width="1027" height="742" alt="image" src="https://github.com/user-attachments/assets/443a7967-8f90-43a2-b15a-26c97c8b1c02" />

---

## Step 5: Verify Versioning Is Enabled

  re-check bucket versioning configuration
  confirm status is Enabled

Command:
  aws s3api get-bucket-versioning \
    --bucket nautilus-s3-9397

Expected Result:
  Status: Enabled

Screenshot:
  <img width="1033" height="734" alt="image" src="https://github.com/user-attachments/assets/526f05f2-8a9c-45a2-a7e6-76d71d91ef26" />

---

## Final Result
- S3 bucket versioning successfully enabled
- Bucket is now protected against accidental deletions and overwrites
- Data recovery capabilities improved without impacting existing objects

---

## Security & Best Practices
- Versioning supports rollback and recovery during incidents
- Critical for compliance, backups, and auditability
- Commonly combined with lifecycle policies and MFA delete in production

---

## Real-World Relevance
This workflow mirrors how DevOps and Cloud Engineers implement
data protection controls for production S3 buckets using CLI-based
configuration rather than manual console actions.

---

## Skills Demonstrated
- AWS S3 administration
- Data protection and recovery concepts
- AWS CLI proficiency
- Cloud operations best practices
