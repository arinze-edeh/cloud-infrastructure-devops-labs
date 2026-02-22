# AWS EC2 Instance with Elastic IP (CLI Provisioning)

## Project Overview
- This project demonstrates how to provision an Amazon EC2 instance using the AWS CLI and associate a static Elastic IP (EIP) to ensure a consistent public endpoint.  
- The setup is performed entirely via CLI commands in the **us-east-1** region, following best practices for reproducibility and validation.

- This lab reflects real-world DevOps workflows including identity verification, resource provisioning, networking configuration, and post-deployment validation.

---

## Architecture Summary

- **Compute**: Amazon EC2 (t2.micro)
- **Networking**: Elastic IP (VPC scope)
- **Region**: us-east-1
- **Provisioning Method**: AWS CLI
- **AMI**: Amazon Linux–based AMI

---

## Step 1: Verify AWS Identity
- Before provisioning resources, confirm the active AWS identity and account context.

`aws sts get-caller-identity`

Expected output confirms:

- IAM User

- AWS Account ID

- Correct permissions context

📸 Screenshot: `aws sts get-caller-identity output`
<img width="1035" height="416" alt="image" src="https://github.com/user-attachments/assets/262ad477-209a-44b9-80ad-e53a4f02e154" />

## Step 2: Launch EC2 Instance

- An EC2 instance is launched using the AWS CLI with tagging applied at creation time.

`aws ec2 run-instances \
  --image-id ami-0c7217cdde317cfec \
  --count 1 \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=datacenter-ec2}]' \
  --region us-east-1`

Key details:

- Instance Type: `t2.micro`

- Tag: `Name=datacenter-ec2`

- Availability Zone: `auto-selected`

📸 Screenshots: `[EC2 run-instances command]`
<img width="952" height="860" alt="image" src="https://github.com/user-attachments/assets/6c93ab9a-9705-45db-9646-dbe35c38f702" />
<img width="1014" height="864" alt="image" src="https://github.com/user-attachments/assets/8d6014df-ffca-45bf-afc5-f42ef7376f4c" />
<img width="956" height="859" alt="image" src="https://github.com/user-attachments/assets/a67d9cbc-3908-463c-9dc0-b434e8c9009d" />
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/0b41dcbe-c63b-4fce-b906-e665566acc8b" />

## Step 3: Allocate Elastic IP

- A VPC-scoped Elastic IP is allocated and tagged for identification.

`aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=datacenter-eip}]' \
  --region us-east-1`

This returns:

- `AllocationId`

- `Public IP address`

📸 Screenshot: `Elastic IP allocation output`
<img width="1023" height="864" alt="image" src="https://github.com/user-attachments/assets/a98b9a32-1f6d-4305-9180-e9aa93108cc4" />

## Step 4: Associate Elastic IP with EC2

- The allocated Elastic IP is associated with the EC2 instance.

`aws ec2 associate-address \
  --allocation-id eipalloc-0cb4e844e5a931a3e \
  --instance-id i-0f6a39c883a6e3cb1 \
  --region us-east-1`

- Successful execution returns an AssociationId.

📸 Screenshot: `Elastic IP association success`
<img width="1024" height="858" alt="image" src="https://github.com/user-attachments/assets/eea56650-352f-4efd-8899-0aa8722a5733" />

##Step 5: Validate Elastic IP Association

- Confirm that the Elastic IP is correctly attached to the instance.

- Verify Elastic IP
`aws ec2 describe-addresses \
  --allocation-ids eipalloc-0cb4e844e5a931a3e \
  --region us-east-1`

- Verify EC2 Public IP
`aws ec2 describe-instances \
  --instance-ids i-0f6a39c883a6e3cb1 \
  --query "Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,PublicIP:PublicIpAddress,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table \
  --region us-east-1`

Expected result:

- Instance state: `running`

- Public IP matches Elastic IP

📸 Screenshots: `[describe-addresses output]` 
<img width="1027" height="855" alt="image" src="https://github.com/user-attachments/assets/b2422731-d84c-4e35-bffd-ac8b1ef6ef23" />

`[describe-instances table output]`
<img width="1032" height="805" alt="image" src="https://github.com/user-attachments/assets/633344bf-73d4-413f-843d-6c5978f03798" />

## Final Outcome

- EC2 instance successfully provisioned via CLI

- Static Elastic IP allocated and associated

- Public endpoint validated and persistent

- Resources correctly tagged and region-scoped

## Tools & Services Used

- AWS CLI

- Amazon EC2

- Elastic IP (VPC)

- IAM (User-based authentication)










