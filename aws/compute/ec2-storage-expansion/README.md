# EC2 Storage Expansion – EBS Volume Creation (AWS CLI)

## Overview
This lab demonstrates how to provision Amazon Elastic Block Store (EBS) storage
to support incremental infrastructure growth in AWS. The objective is to create
a gp3 EBS volume using the AWS CLI, following cloud best practices for storage
planning and resource tagging.

The task reflects a real-world scenario where DevOps teams expand EC2 storage
capacity independently of compute resources.

---

## Lab Context
The Nautilus DevOps team is migrating infrastructure to AWS in incremental phases.
As part of this approach, block storage is provisioned separately to enable
scalability, flexibility, and minimal operational disruption.

---

## Objectives
- Identify a valid Availability Zone in the target region
- Create an EBS volume with defined specifications
- Apply meaningful resource tags
- Verify successful volume creation using AWS CLI

---

## Tools & Services Used
- AWS EC2
- Amazon EBS
- AWS CLI
- Linux Shell

---

## Environment Details
- AWS Region: us-east-1
- Volume Name: nautilus-volume
- Volume Type: gp3
- Volume Size: 2 GiB
- Scope: Availability Zone–specific

---

## Step 1: Verify AWS Region Configuration

SET aws_region = "us-east-1"

VERIFY aws_region is active
📸 Screenshot:

<img width="1034" height="788" alt="image" src="https://github.com/user-attachments/assets/ce3eb6b3-550d-4fa0-9e32-91eba7e61551" />

AWS CLI showing active region (us-east-1)

## Step 2: Identify Available Availability Zones

FETCH availability_zones
SELECT one availability_zone

📸 Screenshot:
<img width="1028" height="827" alt="image" src="https://github.com/user-attachments/assets/762ff340-06d7-4518-9f6a-4f32ff800e03" />

Output showing available AZs in us-east-1

## Step 3: Create EBS Volume

CREATE ebs_volume WITH:
  size = 2 GiB
  type = gp3
  availability_zone = selected_zone
  tag:
    Name = "nautilus-volume"

📸 Screenshot:
<img width="1041" height="863" alt="image" src="https://github.com/user-attachments/assets/5a9883ce-41f5-498a-9c2d-cfa27142e94e" />

AWS CLI output confirming EBS volume creation

Volume ID displayed

## Step 4: Verify Volume State

FETCH volume_details
IF volume.state == "available"
  CONTINUE
ELSE
  INVESTIGATE error

📸 Screenshot:
<img width="1034" height="852" alt="image" src="https://github.com/user-attachments/assets/244fbcbc-d3a4-428d-b014-32d807ad0982" />
Volume status showing "available"

## Step 5: Validate Volume Properties

ASSERT volume.size == 2 GiB
ASSERT volume.type == gp3
ASSERT volume.tag.Name == "nautilus-volume"

📸 Screenshot:
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/594bca28-6d05-451a-a45e-87bf70c62099" />
AWS CLI output confirming volume size, type, and tags

Final Validation Logic
IF volume exists AND
   volume.size == expected_size AND
   volume.type == expected_type AND
   volume.state == "available"
THEN
   RESULT = SUCCESS
ELSE
   RESULT = FAILURE

## Outcome

- EBS volume successfully provisioned

- Storage resource meets all defined requirements

- Volume ready for EC2 attachment and workload usage

## Best Practices Demonstrated

- Independent storage provisioning

- Use of gp3 for cost-effective performance

- Resource tagging for traceability

- CLI-based validation before production use

## Real-World Relevance

- This lab mirrors real production workflows where DevOps engineers provision,
validate, and manage block storage independently to support scalable EC2-based
applications and services.

- Skills Demonstrated

- AWS EC2 storage management

- EBS volume provisioning

- AWS CLI operations

- Infrastructure validation techniques
