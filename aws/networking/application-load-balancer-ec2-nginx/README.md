# AWS Application Load Balancer Web Ingress (EC2 + Nginx)

## Project Overview

- This project documents the deployment of an **internet-facing AWS Application Load Balancer (ALB)** that routes HTTP traffic to an existing **EC2 instance running Nginx** in the **us-east-1** region.

- The objective was to validate **end-to-end Layer 7 traffic flow** from the public internet through the ALB to the backend EC2 instance using **target groups and security group isolation**.

- This setup represents a foundational **web ingress architecture** commonly used in scalable cloud environments.

---

## Architecture Summary

             ┌────────────────────────┐
             │  Application Load       │
             │  Balancer (ALB)         │
             │  HTTP :80               │
             └───────────┬────────────┘
                         │
               ┌─────────▼─────────┐
               │ Target Group       │
               │ HTTP :80           │
               └─────────┬─────────┘
                         │
                ┌────────▼────────┐
                │ EC2 Instance     │
                │ devops-ec2       │
                │ Nginx :80        │
                └─────────────────┘


---

## 🔧 Technologies Used

* Amazon EC2
* Application Load Balancer (ALB)
* Target Groups
* Security Groups
* AWS CLI
* Nginx (Linux/UNIX)

---

## 🌍 Environment Details

| Component | Value |
|--------|------|
| Region | us-east-1 |
| VPC | Default VPC |
| EC2 Instance Name | devops-ec2 |
| Instance Type | t2.micro |
| Load Balancer | devops-alb |
| Target Group | devops-tg |
| ALB Security Group | devops-sg |

---

## 🚀 Implementation Steps

### 1️⃣ Discover Existing EC2 Instance

Identify the backend EC2 instance that will receive traffic.

`aws ec2 describe-instances`

```bash
Instance ID: i-011ae72a16b4398f6

VPC ID: vpc-0c3f56dfe99c83d0d

Subnet: subnet-00b5049732d6fd4eb

Security Group: default
```

📸 Screenshot: EC2 instance details (running state)
<img width="1039" height="805" alt="image" src="https://github.com/user-attachments/assets/9db485cd-3b2d-4b09-b3b9-d049498a51fb" />

### 2️⃣ Confirm Default VPC
aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text

📸 Screenshot: `Default VPC verification` 


### 3️⃣ Create Security Group for ALB

Create a dedicated security group to allow public HTTP traffic.

```bash
aws ec2 create-security-group \
  --group-name devops-sg \
  --description "Security group for devops-alb" \
  --vpc-id vpc-0c3f56dfe99c83d0d

Allow inbound HTTP access:

aws ec2 authorize-security-group-ingress \
  --group-name devops-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

📸 Screenshots: `ALB security group creation` `Inbound rule allowing HTTP :80` 


### 4️⃣ Create Target Group

Define where the ALB will forward incoming traffic.

```bash
aws elbv2 create-target-group \
  --name devops-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0c3f56dfe99c83d0d \
  --target-type instance
```

📸 Screenshot: `Target group configuration`

### 5️⃣ Register EC2 Instance with Target Group
aws elbv2 register-targets \
  --target-group-arn <TARGET_GROUP_ARN> \
  --targets Id=i-011ae72a16b4398f6

📸 Screenshot

[ Screenshot: Target instance registered and healthy ]

### 6️⃣ Retrieve Subnets for ALB
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0c3f56dfe99c83d0d" \
  --query "Subnets[*].SubnetId" \
  --output text

Two subnets were selected to ensure high availability.

📸 Screenshot

[ Screenshot: Subnets in default VPC ]

### 7️⃣ Create Application Load Balancer
aws elbv2 create-load-balancer \
  --name devops-alb \
  --subnets subnet-09d0365261e4c44c6 subnet-00b5049732d6fd4eb \
  --security-groups sg-0f2b9556d703a6a3c

📸 Screenshot

[ Screenshot: ALB successfully created ]

### 8️⃣ Create HTTP Listener

Configure ALB to forward HTTP traffic to the target group.

aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN>

📸 Screenshot

[ Screenshot: Listener forwarding HTTP :80 traffic ]


### 9️⃣ Allow ALB Traffic to EC2

Update EC2 security group to accept traffic only from the ALB.

aws ec2 authorize-security-group-ingress \
  --group-id sg-0d5e8ddf23e6d5ab3 \
  --protocol tcp \
  --port 80 \
  --source-group sg-0f2b9556d703a6a3c

📸 Screenshot

[ Screenshot: EC2 security group allowing ALB source ]


### 🔍 Validation & Health Checks
Retrieve ALB DNS Name
aws elbv2 describe-load-balancers \
  --names devops-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text

Access in browser:

http://devops-alb-1274082417.us-east-1.elb.amazonaws.com
Expected Result

Nginx default welcome page loads successfully

📸 Screenshot

[ Screenshot: Nginx page accessed via ALB DNS ]


### ✅ Validation Checklist
Check	Status
| Check                             | Status |
| --------------------------------- | ------ |
| EC2 instance running              | ✅      |
| ALB created                       | ✅      |
| Target group configured           | ✅      |
| EC2 registered as target          | ✅      |
| Listener forwarding traffic       | ✅      |
| Security group isolation enforced | ✅      |
| Application reachable via ALB     | ✅      |


### 🎯 Final Outcome

- Successfully implemented Layer 7 web ingress

- Public traffic routed securely through ALB

- Backend EC2 protected from direct internet exposure

- Architecture ready for Auto Scaling and HTTPS

- Matches production-grade AWS design patterns

### 🧠 Key Learnings

- ALB operates at Layer 7 (HTTP)

- Target groups decouple ingress from compute

- Security group chaining improves isolation

- ALB DNS endpoint becomes the public entry point

- Multi-subnet ALB enables high availability



<img width="1034" height="828" alt="image" src="https://github.com/user-attachments/assets/a96ff6d9-c1e7-4ef8-80cf-5222400f45b8" />
<img width="1033" height="829" alt="image" src="https://github.com/user-attachments/assets/e9412cb5-c775-4336-9de0-ca9f7f66fcde" />
<img width="1035" height="870" alt="image" src="https://github.com/user-attachments/assets/fb14a97a-917f-4b14-bd57-19e73b2ff7dc" />
<img width="1041" height="368" alt="image" src="https://github.com/user-attachments/assets/8935f55e-8710-4c3d-9334-7b7f34e8b579" />
<img width="1047" height="481" alt="image" src="https://github.com/user-attachments/assets/7dfe32e8-0ebf-4b76-a6d3-2e480b3da011" />
<img width="1034" height="695" alt="image" src="https://github.com/user-attachments/assets/2d437a9b-50f3-4b8a-8f7f-8ce02c2c561e" />
<img width="1039" height="863" alt="image" src="https://github.com/user-attachments/assets/e72780dc-56fa-48f8-9a0a-7308b2d2faee" />
<img width="1036" height="727" alt="image" src="https://github.com/user-attachments/assets/4782d0f9-177e-4c39-9f97-eb559ea407d2" />
<img width="1228" height="862" alt="image" src="https://github.com/user-attachments/assets/6802a510-d17f-4f87-ad93-eb2c49e6a645" />
<img width="1223" height="854" alt="image" src="https://github.com/user-attachments/assets/0589ef65-e44c-4742-8ec8-a196f8be51a1" />
<img width="1221" height="865" alt="image" src="https://github.com/user-attachments/assets/0212a8f0-806b-4eba-80d1-8834d412a0f8" />
<img width="1229" height="861" alt="image" src="https://github.com/user-attachments/assets/605fc323-a727-4452-bed5-c23254f06eb4" />
<img width="1215" height="862" alt="image" src="https://github.com/user-attachments/assets/38b8cdc3-dbf3-45b5-98c0-a909ca30eb99" />

