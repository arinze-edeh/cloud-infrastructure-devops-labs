# EC2 Instance Launch via AWS CLI (Foundational Provisioning)

## Overview
This project demonstrates how to provision a basic Amazon EC2 instance using the AWS CLI.
The lab focuses on understanding the **end-to-end EC2 launch workflow**, including key pair creation,
AMI selection, instance configuration, and validation.

The objective is to showcase **hands-on AWS CLI competence**, which is a core expectation for
Cloud and DevOps roles.

---

## Project Scope
- Launch an EC2 instance using AWS CLI
- Use Amazon Linux AMI
- Configure instance type, key pair, and security group
- Verify successful instance creation
- Follow AWS operational best practices

---

## Tools & Services Used
- AWS EC2
- AWS CLI
- Amazon Linux AMI
- Default VPC & Security Group
- Linux Shell Environment

---

## Environment Details
- REGION = us-east-1
- INSTANCE NAME = devops-ec2
- INSTANCE TYPE = t2.micro
- AMI = Amazon Linux
- KEY PAIR = devops-kp (RSA)
- SECURITY GRP = Default


---

## Step 1: Confirm AWS CLI Configuration

COMMAND:
aws configure list


EXPECTED:
- Credentials configured
- Region set to us-east-1

📸 Screenshot:
<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/64b72aaf-dbc2-4e27-b98e-2ab1bfbbee5c" />

---

## Step 2: Identify Default VPC

COMMAND:
aws ec2 describe-vpcs
--filters Name=isDefault,Values=true
--query "Vpcs[0].VpcId"
--output text


EXPECTED:
- Default VPC ID returned

📸 Screenshot Placeholder:
> Insert screenshot showing default VPC ID output

---

## Step 3: Identify a Subnet in the Default VPC

COMMAND:
aws ec2 describe-subnets
--filters Name=vpc-id,Values=<DEFAULT_VPC_ID>
--query "Subnets[0].SubnetId"
--output text


EXPECTED:
- Subnet ID returned

📸 Screenshot Placeholder:
> Insert screenshot showing subnet ID output

---

## Step 4: Create EC2 Key Pair

COMMAND:
aws ec2 create-key-pair
--key-name devops-kp
--key-type rsa
--query 'KeyMaterial'
--output text > devops-kp.pem


EXPECTED:
- devops-kp.pem created locally

📸 Screenshot Placeholder:
> Insert screenshot showing key pair creation command output

---

## Step 5: Secure the Private Key

COMMAND:
chmod 400 devops-kp.pem


VERIFY:
ls -l devops-kp.pem


EXPECTED:
- File permission: -r--------

📸 Screenshot Placeholder:
> Insert screenshot showing private key permission verification

---

## Step 6: Identify Amazon Linux AMI

COMMAND:
aws ec2 describe-images
--owners amazon
--filters Name=name,Values="amzn2-ami-hvm-*-x86_64-gp2"
--query "Images | sort_by(@, &CreationDate)[-1].ImageId"
--output text


EXPECTED:
- Latest Amazon Linux AMI ID returned

📸 Screenshot Placeholder:
> Insert screenshot showing AMI ID output

---

## Step 7: Launch EC2 Instance

COMMAND:
aws ec2 run-instances
--image-id <AMI_ID>
--instance-type t2.micro
--key-name devops-kp
--subnet-id <SUBNET_ID>
--security-group-ids <DEFAULT_SG_ID>
--tag-specifications
'ResourceType=instance,Tags=[{Key=Name,Value=devops-ec2}]'


EXPECTED:
- Instance ID returned
- Instance state = pending → running

📸 Screenshot Placeholder:
> Insert screenshot showing EC2 instance launch output

---

## Step 8: Verify EC2 Instance Status

COMMAND:
aws ec2 describe-instances
--filters Name=tag:Name,Values=devops-ec2
--query "Reservations[].Instances[].[InstanceId,State.Name]"
--output table


EXPECTED:
- Instance state = running

📸 Screenshot Placeholder:
> Insert screenshot showing instance running status

---

## Final Result
- EC2 instance successfully launched via AWS CLI
- Key pair securely created and managed
- Instance correctly tagged and verified
- All resources deployed in us-east-1

---

## Security & Operational Best Practices
- Private keys are never committed to version control
- Least-privilege default security group used
- Resource tagging applied for traceability
- CLI-based provisioning supports automation readiness

---

## Real-World Relevance
This workflow mirrors how DevOps and Cloud Engineers:
- Provision infrastructure using CLI tools
- Validate resources programmatically
- Operate efficiently without relying on the AWS Console

---

## Skills Demonstrated
- AWS CLI proficiency
- EC2 provisioning fundamentals
- Cloud resource discovery
- Secure key pair management
- Linux permissions handling



<img width="1033" height="813" alt="image" src="https://github.com/user-attachments/assets/6ac265fe-8a00-4a7c-92ad-3d328dd22c22" />
<img width="1028" height="805" alt="image" src="https://github.com/user-attachments/assets/01bc1758-068b-41fa-a006-707a6e0fa560" />
<img width="1029" height="804" alt="image" src="https://github.com/user-attachments/assets/5ae60d3a-b9ab-458e-a216-8e0040f3a111" />
<img width="1035" height="740" alt="image" src="https://github.com/user-attachments/assets/3b694c7e-ee09-49fe-bbef-dec008669969" />
<img width="1026" height="720" alt="image" src="https://github.com/user-attachments/assets/bf048f62-7cd0-48dc-8b8e-80df98149cb3" />
<img width="1031" height="814" alt="image" src="https://github.com/user-attachments/assets/4288311b-ecd2-4e54-902c-849c361203c5" />
<img width="1023" height="839" alt="image" src="https://github.com/user-attachments/assets/fbacab15-f778-4cd4-bb7d-5d354a59f312" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/e3083489-1a78-45f6-be54-0ae227667f25" />
<img width="1019" height="863" alt="image" src="https://github.com/user-attachments/assets/8fd88ca7-3db8-4b39-b17a-45ad9eee663a" />
<img width="1029" height="857" alt="image" src="https://github.com/user-attachments/assets/edfb06e5-55b2-4a6f-be7f-d3a87ce962f2" />
<img width="1026" height="859" alt="image" src="https://github.com/user-attachments/assets/05f015c2-9959-4567-acc0-410b9d51b041" />
<img width="1034" height="865" alt="image" src="https://github.com/user-attachments/assets/e4292377-4e16-4f1d-9b33-44d91816ee1b" />
<img width="1035" height="860" alt="image" src="https://github.com/user-attachments/assets/75b1ca3c-b273-41ba-accc-e38bf2dddab8" />
