# AWS Networking – Subnet Creation in Default VPC (AWS CLI)

## Overview
- This lab demonstrates how to provision a subnet
-  inside an existing AWS default VPC using AWS CLI.
- Subnet creation is a foundational networking task
- required for workload isolation and scalability.

---

## Scenario
-  As part of an incremental cloud migration, a DevOps team needs to prepare network segmentation within AWS before deploying compute resources.
-  A new subnet is required under the default VPC.

---

## Objectives
- Identify the default VPC using AWS CLI
- Inspect existing CIDR allocations
- Create a non-overlapping subnet
- Apply resource tagging
- Verify successful provisioning

---

## Tools & Services Used
- AWS EC2
- AWS VPC
- AWS CLI
- Linux Shell
- IAM (temporary lab credentials)

---

## Environment Details
- Cloud Provider: AWS
- Region: us-east-1
- VPC Type: Default VPC
- Subnet CIDR: /24
- Availability Zone: us-east-1a

---

## Step 1: Verify Active AWS Region
- Ensure all resources are created in the intended region

aws configure get region

Expected Output:
- us-east-1

📸 Screenshot: Region verification
<img width="1035" height="861" alt="image" src="https://github.com/user-attachments/assets/68cb9cd9-87cd-40cd-9900-a5645d558212" />
---

## Step 2: Identify the Default VPC
- Query all VPCs and filter for the default VPC

aws ec2 describe-vpcs \
  --query "Vpcs[*].{VpcId:VpcId,Cidr:CidrBlock,IsDefault:IsDefault}" \
  --output table

// Store the VPC ID where IsDefault = true

📸 Screenshot: Default VPC details
<img width="1035" height="861" alt="image" src="https://github.com/user-attachments/assets/68cb9cd9-87cd-40cd-9900-a5645d558212" />

---

## Step 3: Inspect Existing Subnets
- Prevent CIDR conflicts by reviewing existing subnets

aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=<DEFAULT_VPC_ID> \
  --query "Subnets[*].CidrBlock" \
  --output table

- Select an unused CIDR block within the VPC range

📸 Screenshot: Existing subnet CIDRs

---

## Step 4: Create the Subnet
- Provision a new subnet using a valid CIDR block

aws ec2 create-subnet \
  --vpc-id <DEFAULT_VPC_ID> \
  --cidr-block 172.31.128.0/24 \
  --availability-zone us-east-1a

📸 Screenshot: Subnet creation output

---

## Step 5: Tag the Subnet
- Assign a Name tag for identification and lifecycle management

aws ec2 create-tags \
  --resources <SUBNET_ID> \
  --tags Key=Name,Value=devops-subnet

📸 Screenshot: Subnet tagging confirmation

---

## Step 6: Final Verification
- Confirm subnet existence, CIDR block, and AZ placement

aws ec2 describe-subnets \
  --filters Name=tag:Name,Values=devops-subnet \
  --query "Subnets[*].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}" \
  --output table

📸 Screenshot: Final verification output
<img width="1034" height="861" alt="Screenshot 2026-02-01 045806" src="https://github.com/user-attachments/assets/0c970cc3-0f01-488a-93cd-2d0c8c3036c7" />

---

## Result
- Default VPC successfully identified
- CIDR conflicts avoided through validation
- Subnet created and tagged correctly
- Network layer ready for compute deployment

---

## Real-World Relevance
- This task mirrors real-world cloud migrations
- where networking must be established
- before EC2, RDS, or container workloads.
- Proper CIDR planning prevents production outages.

---

## Skills Demonstrated
- AWS VPC & subnet management
- CIDR planning and validation
- AWS CLI proficiency
- Cloud networking fundamentals

