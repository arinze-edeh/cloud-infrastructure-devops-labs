# EC2 Nginx Web Server Deployment via AWS CLI

## Project Overview

This project documents a **CLI-driven provisioning of an Ubuntu EC2 instance** configured as a public-facing **Nginx web server**. The setup uses AWS CLI commands, security groups, and EC2 user data to achieve a fully automated and reproducible infrastructure workflow in the **us-east-1** region. This aligns with DevOps best practices for infrastructure automation and validation.

---

## Architecture Summary

```
Internet
   |
Port 80 (HTTP)
   |
AWS Security Group (xfusion-web-sg)
   |
EC2 Instance (Ubuntu)
   |
Nginx Web Server
```

---

## Prerequisites

```
AWS CLI installed
Valid AWS credentials configured
Region set to us-east-1
```

---

## Step 1: Validate AWS Identity

```
EXECUTE aws sts get-caller-identity
CONFIRM correct Account ID and IAM user
```

### Screenshot:

<img width="1029" height="531" alt="image" src="https://github.com/user-attachments/assets/aef74ccc-011d-4f36-9670-7090f92c1f20" />

---

## Step 2: Identify Default VPC

```
EXECUTE aws ec2 describe-vpcs
CONFIRM default VPC is available
NOTE VPC ID for security group creation
```

### Screenshot:

###

---

## Step 3: Create Security Group

```
EXECUTE aws ec2 create-security-group
SET group-name = xfusion-web-sg
SET description = Security group for xfusion-ec2 Nginx server
ASSOCIATE with default VPC
STORE Security Group ID
```

### Screenshot:

*

---

## Step 4: Allow HTTP Ingress

```
EXECUTE aws ec2 authorize-security-group-ingress
ALLOW TCP port 80
SOURCE = 0.0.0.0/0
```

### Screenshot:

**

---

## Step 5: Create User Data Script

```
CREATE file userdata.sh
ADD the following:

#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
```

### Screenshot:

---

---

## Step 6: Launch EC2 Instance

```
EXECUTE aws ec2 run-instances
SET AMI = Ubuntu
SET instance-type = t2.micro
ATTACH security group xfusion-web-sg
ATTACH user-data file userdata.sh
TAG instance Name=xfusion-ec2
```

### Screenshot:

##

---

## Step 7: Retrieve Public IP Address

```
EXECUTE aws ec2 describe-instances
FILTER by tag Name=xfusion-ec2
EXTRACT PublicIpAddress
```

### Screenshot:

###

---

## Step 8: Validate Nginx Deployment

```
EXECUTE curl -I http://<public-ip>
CONFIRM HTTP/1.1 200 OK
CONFIRM Server: nginx
```

### Screenshot:

#

---

## Validation Checklist

```
[ ] EC2 instance name is xfusion-ec2
[ ] Instance running in us-east-1
[ ] Security group allows HTTP (80)
[ ] User data executed successfully
[ ] Nginx service active
[ ] Web server reachable from internet
```

---

## Operational Notes

```
Deployment performed entirely via AWS CLI
User data ensures idempotent server bootstrap
Security group enforces minimal public access
```

---

## Cleanup (Optional)

```
TERMINATE EC2 instance
DELETE security group
REMOVE key pair if unused
```






<img width="1034" height="809" alt="image" src="https://github.com/user-attachments/assets/b3c344a2-e8d0-4172-9192-d9143cdc0ffd" />
<img width="1030" height="863" alt="image" src="https://github.com/user-attachments/assets/4fa86e2f-b54a-48f3-a2aa-3283a96b0484" />
<img width="1030" height="640" alt="image" src="https://github.com/user-attachments/assets/d38ece55-85ec-40d4-96e7-954c7576b47f" />
<img width="1030" height="620" alt="image" src="https://github.com/user-attachments/assets/5c89fc1b-48ed-4015-a4e6-3492cbd52859" />
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/5f62c78b-1fa6-4672-acd6-a3d477d49bba" />
<img width="1030" height="865" alt="image" src="https://github.com/user-attachments/assets/081e38d3-5a83-4c03-b3bd-15b072496700" />
<img width="1025" height="859" alt="image" src="https://github.com/user-attachments/assets/3a1fc5dd-a25a-41a8-b0a7-8d1ea35df23d" />
<img width="1028" height="859" alt="image" src="https://github.com/user-attachments/assets/e31f38fa-c820-452f-bf13-1adc28021f08" />
<img width="1029" height="547" alt="image" src="https://github.com/user-attachments/assets/a52ae395-0248-405d-b52f-967234e880a2" />
<img width="1032" height="357" alt="image" src="https://github.com/user-attachments/assets/81672a03-be32-4869-a386-595f7ec949cb" />


