# AWS Public Infrastructure Baseline (VPC + Internet-Facing EC2)

## Project Overview
This project documents the creation of a **public AWS infrastructure baseline** designed to support internet-facing workloads. The baseline includes a custom VPC, a public subnet with automatic public IP assignment, an Internet Gateway, routing configuration, a security group allowing SSH access, and a publicly accessible EC2 instance.  

All resources are provisioned using the AWS CLI in the **us-east-1** region.

---

## Scope and Objectives
- Establish a non-default public VPC
- Enable outbound and inbound internet connectivity
- Enforce explicit routing via Internet Gateway
- Provision a minimal EC2 compute resource for validation
- Serve as a reusable **network foundation** for future workloads

---

## Architecture Overview

### Logical Components
- VPC (10.0.0.0/16)
- Public Subnet (10.0.1.0/24)
- Internet Gateway
- Route Table with 0.0.0.0/0 route
- Security Group (SSH access)
- EC2 Instance (t2.micro)
  
---

## Environment and Preconditions

### Region
```
AWS_REGION = us-east-1
Identity Verification
aws sts get-caller-identity
```
Screenshot: `AWS STS Identity Verification`
<img width="1034" height="530" alt="image" src="https://github.com/user-attachments/assets/a1ee7e16-c8dc-4fb3-8149-23cdff0c7210" />

## Implementation Steps

## Step 1: Create Public VPC
```
create vpc
cidr_block = 10.0.0.0/16
name = devops-pub-vpc
enable_dns_support = true
enable_dns_hostnames = true
```
Screenshot: `VPC Created Successfully`
<img width="1028" height="839" alt="image" src="https://github.com/user-attachments/assets/43e6bc23-6db9-4a9d-aa67-6e957d5db9bc" />

## Step 2: Create Internet Gateway

```
create internet_gateway
name = devops-pub-igw
```
Screenshot: `Internet Gateway Created`
<img width="1032" height="652" alt="image" src="https://github.com/user-attachments/assets/a1364c50-6851-45f4-9211-3d01a5850a3c" />

## Step 3: Attach Internet Gateway to VPC
```
attach internet_gateway
to vpc = devops-pub-vpc
```
Screenshot: `Internet Gateway Attached to VPC`

<img width="1034" height="451" alt="image" src="https://github.com/user-attachments/assets/22631e7f-2c25-49b0-9d6e-e79a6a4c855a" />

## Step 4: Create Public Subnet
```
create subnet
vpc = devops-pub-vpc
cidr_block = 10.0.1.0/24
availability_zone = us-east-1e
name = devops-pub-subnet
```
Screenshot: `Subnet Created`

## Step 5: Enable Auto-Assign Public IP on Subnet
```
modify subnet attribute
subnet = devops-pub-subnet
map_public_ip_on_launch = true
```
Screenshot: `Public IP Auto-Assignment Enabled`

## Step 6: Create Route Table
```
create route_table
vpc = devops-pub-vpc
name = devops-pub-rt
```
Screenshot: `Route Table Created`

## Step 7: Add Internet Route
```
add route
route_table = devops-pub-rt
destination = 0.0.0.0/0
target = internet_gateway
```
Screenshot: `Default Route Added`

## Step 8: Associate Route Table with Public Subnet
```
associate route_table
route_table = devops-pub-rt
subnet = devops-pub-subnet
```
Screenshot: `Route Table Association`

## Step 9: Create Security Group (SSH Access)
```
create security_group
name = devops-ssh-sg
vpc = devops-pub-vpc
description = allow ssh access
```
Screenshot: `Security Group Created`

## Step 10: Allow Inbound SSH
```
authorize ingress
security_group = devops-ssh-sg
protocol = tcp
port = 22
source = 0.0.0.0/0
```
Screenshot: `SSH Rule Configured`

## Step 11: Launch EC2 Instance
```
run ec2_instance
name = devops-pub-ec2
ami = amazon_linux_ami
instance_type = t2.micro
subnet = devops-pub-subnet
security_group = devops-ssh-sg
assign_public_ip = true
```
Screenshot: `EC2 Instance Launch Initiated`

## Step 12: Validate Public Accessibility
```
describe ec2_instance
filter = name:devops-pub-ec2
retrieve public_ip
```
Screenshot: `Public IP Assigned`

## Validation Checklist

- VPC is non-default and active

- Subnet auto-assigns public IPs

- Route table contains 0.0.0.0/0 → IGW

- EC2 instance has a public IP

- SSH port 22 reachable from the internet

Screenshot: End-to-End Validation

## Key Learnings

- Public subnets require explicit routing to an Internet Gateway

- Public IP assignment must be enabled at subnet or instance level

- Network infrastructure should be provisioned before compute

- A baseline VPC simplifies future scaling and governance

## Cleanup Procedure
- terminate ec2_instance
- delete security_group
- disassociate and delete route_table
- detach and delete internet_gateway
- delete subnet
- delete vpc

## Status
- PROJECT_STATUS = `completed`
- BASELINE_READY_FOR_EXTENSION = `true`





<img width="1031" height="857" alt="image" src="https://github.com/user-attachments/assets/129745cc-9ef4-4bc6-b506-a0b90a49f9ec" />
<img width="1036" height="515" alt="image" src="https://github.com/user-attachments/assets/021e1a60-62a2-4129-959d-5166a72a6e29" />
<img width="1032" height="705" alt="image" src="https://github.com/user-attachments/assets/97adc556-204d-4c3c-a6b8-55275af9e0a2" />
<img width="1029" height="787" alt="image" src="https://github.com/user-attachments/assets/c5b00172-c77e-4e07-a372-be8995c44522" />
<img width="1035" height="418" alt="image" src="https://github.com/user-attachments/assets/2b204198-b322-43d7-891b-ac1d4928e5c6" />
<img width="1030" height="537" alt="image" src="https://github.com/user-attachments/assets/c9868ac8-2178-47e5-ab6f-f3330eb537bd" />
<img width="1036" height="595" alt="image" src="https://github.com/user-attachments/assets/4e5cabc6-3ffa-4683-b341-7879a0df17a9" />
<img width="1029" height="854" alt="image" src="https://github.com/user-attachments/assets/ddafa25b-4f06-4bd5-9050-2c9d76f80f98" />
<img width="1020" height="863" alt="image" src="https://github.com/user-attachments/assets/686f19e0-2e38-4430-be8c-d42f00dea3ae" />
<img width="1030" height="862" alt="image" src="https://github.com/user-attachments/assets/022da898-e348-4b82-ac52-88a9f6a028ff" />
<img width="1035" height="482" alt="image" src="https://github.com/user-attachments/assets/f8c6a09f-c37c-49a0-861f-42c2b979936c" />
<img width="1040" height="801" alt="image" src="https://github.com/user-attachments/assets/9bad9d59-f2d2-4505-bc54-675b3850df24" />

