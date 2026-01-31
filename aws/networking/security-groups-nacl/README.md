# AWS Security Group Configuration (EC2 Access Control)

## Overview

This lab demonstrates **practical AWS networking security**, focusing on **firewall rules**, **least-privilege access**, and **CLI-based cloud operations**.

---

## Architecture Context

ENVIRONMENT:
- Cloud Provider = AWS
- Region = us-east-1
- Network = Default VPC
- Resource Type = Security Group
- Access Method = AWS CLI


---

## Step 1: Identify the Target VPC


ACTION:
- Query AWS to identify the default VPC
COMMAND:
- aws ec2 describe-vpcs
FILTER isDefault = true
OUTPUT:
- VPC_ID retrieved for security group association


📸 **Screenshot**  
<img width="1048" height="906" alt="image" src="https://github.com/user-attachments/assets/11847b2f-bfdf-48b4-855a-d74363241635" />

---

## Step 2: Create Security Group


ACTION:
 Create a security group inside the default VPC
INPUT PARAMETERS:
- Group Name = nautilus-sg
- Description = "Security group for Nautilus App Servers"
- VPC_ID = <default-vpc-id>
COMMAND:
- aws ec2 create-security-group


EXPECTED RESULT:


OUTPUT:
SecurityGroupId generated successfully


📸 **Screenshot**  
<img width="1048" height="906" alt="image" src="https://github.com/user-attachments/assets/11847b2f-bfdf-48b4-855a-d74363241635" />

---

## Step 3: Configure Inbound Rules (Firewall Rules)


RULE 1:
Protocol = HTTP
Port = 80
Source = 0.0.0.0/0
Purpose = Allow public web traffic

RULE 2:
Protocol = SSH
Port = 22
Source = 0.0.0.0/0
Purpose = Enable administrative access


COMMANDS:


aws ec2 authorize-security-group-ingress


📸 **Screenshot**  
<img width="1030" height="909" alt="image" src="https://github.com/user-attachments/assets/5618d954-e849-4621-9fbd-63eef140042d" />

---

## Step 4: Verify Security Group Configuration


ACTION:
Retrieve and validate security group rules
COMMAND:
aws ec2 describe-security-groups
VALIDATION:
Confirm:
- HTTP (80) rule exists
- SSH (22) rule exists
- Correct CIDR ranges applied


📸 **Screenshot**  
<img width="1075" height="901" alt="image" src="https://github.com/user-attachments/assets/02d93e47-5a6e-4c3a-a24b-279a0f6f09eb" />
<img width="1044" height="880" alt="image" src="https://github.com/user-attachments/assets/17853506-e6a6-44a3-bd10-a3113cdf4b4c" />
<img width="1060" height="892" alt="image" src="https://github.com/user-attachments/assets/8aeca849-6020-4ae5-b0a3-14dbb7435dec" />

---

## Result


RESULT:
- EC2 traffic is securely controlled via Security Group
- Only required ports are exposed
- Security group is ready for EC2 instance attachment


---

## Security Best Practices Applied


- Network access controlled at the VPC level
- Explicit inbound rules defined
- Security group scoped to application use case
- CLI used for repeatable, auditable configuration


> NOTE: In production environments, SSH access should be restricted to trusted IP ranges instead of `0.0.0.0/0`.

---

## Real-World Relevance


USE CASES:
- Securing web application servers
- Enforcing network-level access controls
- Preparing infrastructure for scalable EC2 deployments


This mirrors **real DevOps and Cloud Engineer workflows** used in production AWS environments.

---

## Skills Demonstrated


- AWS Networking Fundamentals
- Security Groups & Firewall Rules
- AWS CLI Proficiency
- Cloud Security Best Practices
- Infrastructure Validation & Troubleshooting


---

## Recruiter Summary


This lab proves hands-on experience with:
- AWS network security
- CLI-driven infrastructure management
- Production-aligned cloud access control


---
