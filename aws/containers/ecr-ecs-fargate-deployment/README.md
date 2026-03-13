# Containerized Application Deployment on AWS ECS with Fargate and ECR

[![AWS](https://img.shields.io/badge/AWS-ECS%20%7C%20ECR%20%7C%20Fargate-FF9900?style=flat-square&logo=amazon-aws)](https://aws.amazon.com)
[![Docker](https://img.shields.io/badge/Docker-nginx%3Aalpine-2496ED?style=flat-square&logo=docker)](https://www.docker.com)
[![Infrastructure](https://img.shields.io/badge/Infrastructure-Production--Ready-green?style=flat-square)](https://aws.amazon.com/ecs)
[![License](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Architecture](#solution-architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1 - IAM Role Setup](#step-1---iam-role-setup)
  - [Step 2 - ECR Repository Creation](#step-2---ecr-repository-creation)
  - [Step 3 - Docker Image Build and Push](#step-3---docker-image-build-and-push)
  - [Step 4 - ECS Cluster Provisioning](#step-4---ecs-cluster-provisioning)
  - [Step 5 - CloudWatch Log Group](#step-5---cloudwatch-log-group)
  - [Step 6 - Task Definition Registration](#step-6---task-definition-registration)
  - [Step 7 - Network Configuration](#step-7---network-configuration)
  - [Step 8 - ECS Service Deployment](#step-8---ecs-service-deployment)
  - [Step 9 - Verification and Access](#step-9---verification-and-access)
- [Resource Summary](#resource-summary)
- [Cleanup](#cleanup)
- [Troubleshooting](#troubleshooting)

---

## Overview

This runbook documents the end-to-end deployment of a containerized nginx web application using **Amazon Elastic Container Registry (ECR)** for private image storage and **Amazon Elastic Container Service (ECS)** with the **AWS Fargate** serverless compute engine. The deployment achieves a publicly accessible, fully managed containerized workload without provisioning or managing any underlying EC2 infrastructure.

---

## Problem Statement

The Nautilus DevOps team required a production-grade container deployment pipeline with the following non-negotiable requirements:

| Requirement | Constraint |
|---|---|
| Container image storage | Private registry, scan on push enabled |
| Compute model | Serverless (no EC2 instance management) |
| Application | nginx-based web server from `/root/pyapp` Dockerfile |
| Image tag strategy | Mutable, tagged `latest` |
| Cluster name | `xfusion-cluster` |
| Service name | `xfusion-service` |
| Task definition name | `xfusion-taskdefinition` |
| Minimum running tasks | 1 |
| Observability | Centralized CloudWatch logging |

**Core Challenge:** The team needed to go from a raw Dockerfile to a publicly accessible, running ECS Fargate service with proper IAM permissions, networking, logging, and image scanning configured in a single reproducible workflow.

---

## Solution Architecture

```
Developer Workstation (aws-client host)
        |
        | docker build / docker push
        v
+-------------------------+
|   Amazon ECR            |
|   xfusion-ecr           |  <-- Private repo, scan on push, AES256 encryption
|   (us-east-1)           |
+-------------------------+
        |
        | image pull at task start
        v
+--------------------------------------------+
|   Amazon ECS Cluster: xfusion-cluster      |
|   Capacity Provider: FARGATE               |
|                                            |
|   +--------------------------------------+ |
|   |  ECS Service: xfusion-service        | |
|   |  Launch Type: FARGATE                | |
|   |  Desired Count: 1                    | |
|   |                                      | |
|   |  +--------------------------------+  | |
|   |  |  Task: xfusion-taskdefinition  |  | |
|   |  |  CPU: 256 | Memory: 512 MB     |  | |
|   |  |  Container Port: 80/tcp        |  | |
|   |  |  Public IP: ENABLED            |  | |
|   |  +--------------------------------+  | |
|   +--------------------------------------+ |
+--------------------------------------------+
        |
        | awslogs driver
        v
+-------------------------+
|   CloudWatch Logs       |
|   /ecs/xfusion-         |
|   taskdefinition        |
+-------------------------+
        |
        | Security Group: sg-06cc7634e44dc3fbe
        | Inbound: TCP 80 from 0.0.0.0/0
        v
Public Internet --> http://107.23.131.108 (nginx HTTP 200 OK)
```

---

## Prerequisites

- AWS CLI v2 configured with appropriate IAM permissions
- Docker Engine installed and running on the host
- IAM user with permissions for: `iam:*`, `ecr:*`, `ecs:*`, `ec2:*`, `logs:*`
- AWS Region: `us-east-1`
- Account ID: Available via `aws sts get-caller-identity`

---

## Implementation

### Step 1 - IAM Role Setup

**Problem:** ECS tasks running on Fargate require an execution role to pull images from ECR and ship logs to CloudWatch. Without this role, task launches fail at the image pull stage.

**Resolution:** Create the `ecsTaskExecutionRole` with the AWS-managed `AmazonECSTaskExecutionRolePolicy` attached.

```bash
# Create the trust policy document
cat > /root/ecs-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file:///root/ecs-trust-policy.json

# Attach the managed execution policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Verify attachment
aws iam list-attached-role-policies \
  --role-name ecsTaskExecutionRole
```

**Expected Output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonECSTaskExecutionRolePolicy",
            "PolicyArn": "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
        }
    ]
}
```

#### Screenshots - Step 1: IAM Role Creation and Policy Attachment
> *Terminal output showing `ecsTaskExecutionRole` successfully created and `AmazonECSTaskExecutionRolePolicy` attached. The `list-attached-role-policies` response confirms the policy ARN is bound to the role.*
<img width="1032" height="840" alt="image" src="https://github.com/user-attachments/assets/d93b83a0-7bc9-4ffb-8e8e-96fdad9f51dc" />

<img width="1035" height="806" alt="image" src="https://github.com/user-attachments/assets/39b0cb0c-34cb-4adf-bb0c-8e1707bc5515" />


---

### Step 2 - ECR Repository Creation

**Problem:** A private container registry is required to securely store and version Docker images, with vulnerability scanning enabled to meet security compliance requirements.

**Resolution:** Create a private ECR repository named `xfusion-ecr` with `scanOnPush=true` and mutable tags.

```bash
aws ecr create-repository \
  --repository-name xfusion-ecr \
  --region us-east-1 \
  --image-scanning-configuration scanOnPush=true \
  --image-tag-mutability MUTABLE
```

**Verify the repository URI:**
```bash
aws ecr describe-repositories \
  --repository-names xfusion-ecr \
  --region us-east-1 \
  --query 'repositories[0].repositoryUri' \
  --output text
# Output: 692699826578.dkr.ecr.us-east-1.amazonaws.com/xfusion-ecr
```

#### Screenshot - Step 2: ECR Private Repository Created
> *AWS CLI output showing the `xfusion-ecr` repository creation response, including `repositoryUri`, `scanOnPush: true`, `imageTagMutability: MUTABLE`, and `encryptionType: AES256`.*

<img width="1028" height="724" alt="image" src="https://github.com/user-attachments/assets/9f0cbbb9-8930-4333-832e-c363b7e3ed94" />

---

### Step 3 - Docker Image Build and Push

**Problem:** The application Dockerfile exists locally at `/root/pyapp` and must be built, tagged with the ECR registry URI, and pushed to the private repository so ECS can pull it at runtime.

**Resolution:** Authenticate Docker to ECR, build the image, tag it appropriately, and push.

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 \
  | docker login \
  --username AWS \
  --password-stdin 692699826578.dkr.ecr.us-east-1.amazonaws.com

# Navigate to the application directory
cd /root/pyapp

# Build the Docker image
docker build -t xfusion-ecr .

# Tag the image with the full ECR URI
docker tag xfusion-ecr:latest \
  692699826578.dkr.ecr.us-east-1.amazonaws.com/xfusion-ecr:latest

# Push to ECR
docker push \
  692699826578.dkr.ecr.us-east-1.amazonaws.com/xfusion-ecr:latest
```

**Verify the image exists in ECR:**
```bash
aws ecr list-images \
  --repository-name xfusion-ecr \
  --region us-east-1
```

**Expected Output:**
```json
{
    "imageIds": [
        {
            "imageDigest": "sha256:b0ad7553bb399922c9ae030cb57bcc186352ece8acd8337806d53c79a4b47b91",
            "imageTag": "latest"
        }
    ]
}
```

#### Screenshots - Step 3: Docker Build, Tag, and Push to ECR
> *Terminal showing `Login Succeeded`, all Docker build steps completing with `Successfully built 92fa306c3077` and `Successfully tagged xfusion-ecr:latest`, followed by all image layers pushed and the `latest` digest confirmed in `ecr list-images`.*

<img width="1024" height="772" alt="image" src="https://github.com/user-attachments/assets/f515dadb-0c13-4835-83bd-d6cc98880ee5" />
<img width="1028" height="592" alt="image" src="https://github.com/user-attachments/assets/6866612a-43a1-4570-8cbf-3ba358019dd6" />
<img width="1033" height="370" alt="image" src="https://github.com/user-attachments/assets/b9344e12-aba8-4bc9-ad32-73deef00fa98" />
<img width="1028" height="557" alt="image" src="https://github.com/user-attachments/assets/cbb4444d-ccb1-44c4-9569-132c5d183bc3" />

---

### Step 4 - ECS Cluster Provisioning

**Problem:** A logical grouping construct is needed to house ECS services and tasks. The cluster must use Fargate as the capacity provider to eliminate EC2 instance management overhead.

**Resolution:** Create the `xfusion-cluster` ECS cluster with FARGATE as the default capacity provider.

```bash
aws ecs create-cluster \
  --cluster-name xfusion-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1 \
  --region us-east-1
```

**Expected Output (key fields):**
```json
{
    "cluster": {
        "clusterName": "xfusion-cluster",
        "status": "ACTIVE",
        "capacityProviders": ["FARGATE"]
    }
}
```

#### Screenshot - Step 4: ECS Cluster Active with Fargate Capacity Provider
> *AWS CLI response confirming `xfusion-cluster` in `ACTIVE` status, `capacityProviders: ["FARGATE"]`, and `defaultCapacityProviderStrategy` showing weight 1.*

<img width="1036" height="731" alt="image" src="https://github.com/user-attachments/assets/8c027838-23f0-4d5f-b44a-226b930abf25" />

---

### Step 5 - CloudWatch Log Group

**Problem:** Without a pre-existing CloudWatch log group, the ECS task using the `awslogs` log driver will fail to start because it cannot deliver logs.

**Resolution:** Create the log group before registering the task definition.

```bash
aws logs create-log-group \
  --log-group-name /ecs/xfusion-taskdefinition \
  --region us-east-1
```

#### Screenshot - Step 5: CloudWatch Log Group Created
> *Terminal showing the `aws logs create-log-group` command completing with no error (silent success), confirming `/ecs/xfusion-taskdefinition` log group is provisioned and ready to receive container logs.*

<img width="1030" height="732" alt="image" src="https://github.com/user-attachments/assets/c13e5485-d6f0-41d1-8a83-04b50b44958a" />

---

### Step 6 - Task Definition Registration

**Problem:** ECS requires a task definition that specifies the container image, resource allocations, port mappings, execution role, and logging configuration before a service can be created.

**Resolution:** Register `xfusion-taskdefinition` with 256 CPU units, 512 MB memory, port 80 exposed, and `awslogs` configured.

```bash
cat > /root/xfusion-taskdefinition.json << 'EOF'
{
  "family": "xfusion-taskdefinition",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::692699826578:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "xfusion-container",
      "image": "692699826578.dkr.ecr.us-east-1.amazonaws.com/xfusion-ecr:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/xfusion-taskdefinition",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition \
  --cli-input-json file:///root/xfusion-taskdefinition.json \
  --region us-east-1
```

**Verify registration:**
```
Task Definition ARN: arn:aws:ecs:us-east-1:692699826578:task-definition/xfusion-taskdefinition:1
```

#### Screenshots - Step 6: Task Definition Registered
> *Terminal output showing the full `register-task-definition` response with `taskDefinitionArn` ending in `xfusion-taskdefinition:1`, container definition fields including `image`, `portMappings`, `logConfiguration`, and `status: ACTIVE`.*

<img width="1030" height="732" alt="image" src="https://github.com/user-attachments/assets/c13e5485-d6f0-41d1-8a83-04b50b44958a" />
<img width="1019" height="850" alt="image" src="https://github.com/user-attachments/assets/de4103eb-1c8e-4f6d-ac48-01a1c49c9b6d" />
<img width="1029" height="860" alt="image" src="https://github.com/user-attachments/assets/bccaa09e-7ca0-48c6-8538-77071dd40d52" />

---

### Step 7 - Network Configuration

**Problem:** The Fargate task requires a subnet and security group for `awsvpc` networking. Port 80 must be open to inbound traffic for the application to be publicly accessible.

**Resolution:** Retrieve default subnet and security group IDs, then add an inbound rule for TCP 80.

```bash
# Get the default subnet in the first availability zone
aws ec2 describe-subnets \
  --filters "Name=defaultForAz,Values=true" \
  --region us-east-1 \
  --query 'Subnets[0].SubnetId' \
  --output text
# Output: subnet-0e4c66f3bebb8e200

# Get the default security group ID
aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=default" \
  --region us-east-1 \
  --query 'SecurityGroups[0].GroupId' \
  --output text
# Output: sg-06cc7634e44dc3fbe

# Allow inbound HTTP traffic on port 80
aws ec2 authorize-security-group-ingress \
  --group-id sg-06cc7634e44dc3fbe \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

#### Screenshot - Step 7: Security Group Inbound Rule Added
> *Terminal output from `authorize-security-group-ingress` showing `Return: true` and the new rule `sgr-079ec3805c6174468` with `IpProtocol: tcp`, `FromPort: 80`, `ToPort: 80`, and `CidrIpv4: 0.0.0.0/0`.*

<img width="1029" height="750" alt="image" src="https://github.com/user-attachments/assets/8cc6eebb-0702-41b3-8b7b-130e4a4e2fe6" />

---

### Step 8 - ECS Service Deployment

**Problem:** A task definition alone does not run containers. An ECS service is required to maintain a desired count of running tasks and handle task replacement on failure.

**Resolution:** Create `xfusion-service` on `xfusion-cluster` with `desiredCount=1`, Fargate launch type, and public IP assignment enabled.

```bash
aws ecs create-service \
  --cluster xfusion-cluster \
  --service-name xfusion-service \
  --task-definition xfusion-taskdefinition \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0e4c66f3bebb8e200],securityGroups=[sg-06cc7634e44dc3fbe],assignPublicIp=ENABLED}" \
  --region us-east-1
```

**Verify the service reached steady state:**
```bash
aws ecs describe-services \
  --cluster xfusion-cluster \
  --services xfusion-service \
  --region us-east-1 \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}'
```

**Expected Output:**
```json
{
    "Status": "ACTIVE",
    "Running": 1,
    "Desired": 1
}
```

#### Screenshots - Step 8: ECS Service Deployed and at Steady State
> *CLI output from `describe-services` confirming `xfusion-service` with `Status: ACTIVE`, `Running: 1`, and `Desired: 1`, indicating the service has reached its desired task count successfully.*

<img width="1029" height="859" alt="image" src="https://github.com/user-attachments/assets/39a8465e-64f0-40fe-a755-6b26edb283f3" />
<img width="1027" height="861" alt="image" src="https://github.com/user-attachments/assets/af0f7b69-6854-4b21-b98b-c9b3411a9e51" />
<img width="1030" height="450" alt="image" src="https://github.com/user-attachments/assets/4f297978-b94c-4809-89df-cfb93ddb1655" />

---

### Step 9 - Verification and Access

**Problem:** After deployment, the public IP of the running Fargate task must be retrieved and the application confirmed reachable over HTTP.

**Resolution:** Retrieve the task's ENI attachment and query the associated public IP, then validate with an HTTP request.

```bash
# List running tasks
aws ecs list-tasks \
  --cluster xfusion-cluster \
  --service-name xfusion-service \
  --region us-east-1

# Extract the task ARN and retrieve the public IP
TASK_ARN="arn:aws:ecs:us-east-1:692699826578:task/xfusion-cluster/3b039902e0b64d9082fef4c93d141e53"

ENI_ID=$(aws ecs describe-tasks \
  --cluster xfusion-cluster \
  --tasks $TASK_ARN \
  --region us-east-1 \
  --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' \
  --output text)

aws ec2 describe-network-interfaces \
  --network-interface-ids $ENI_ID \
  --region us-east-1 \
  --query 'NetworkInterfaces[0].Association.PublicIp' \
  --output text
# Output: 107.23.131.108

# Validate HTTP 200 response
curl -I http://107.23.131.108
```

**Expected HTTP Response:**
```
HTTP/1.1 200 OK
Server: nginx/1.29.6
Date: Fri, 13 Mar 2026 01:57:52 GMT
Content-Type: text/html
Content-Length: 255
```

#### Screenshot - Step 9: HTTP 200 OK - Application Publicly Accessible
> *Terminal showing the public IP `107.23.131.108` resolved from the ENI, followed by `curl -I http://107.23.131.108` returning `HTTP/1.1 200 OK` with `Server: nginx/1.29.6`, confirming the containerized application is live and reachable.*

<img width="1032" height="787" alt="image" src="https://github.com/user-attachments/assets/d16739d8-0cc1-4199-8d28-d5094cf94dee" />

---

## Resource Summary

| Resource | Name / ID | Type |
|---|---|---|
| IAM Role | `ecsTaskExecutionRole` | Execution Role |
| ECR Repository | `xfusion-ecr` | Private Registry |
| ECR Image URI | `692699826578.dkr.ecr.us-east-1.amazonaws.com/xfusion-ecr:latest` | Container Image |
| ECS Cluster | `xfusion-cluster` | Fargate Cluster |
| ECS Task Definition | `xfusion-taskdefinition:1` | Task Definition (256 CPU / 512 MB) |
| ECS Service | `xfusion-service` | Replica Service |
| CloudWatch Log Group | `/ecs/xfusion-taskdefinition` | Log Group |
| VPC Subnet | `subnet-0e4c66f3bebb8e200` | Default Public Subnet |
| Security Group | `sg-06cc7634e44dc3fbe` | Default SG (TCP 80 open) |
| Public IP | `107.23.131.108` | Fargate Task ENI |
| AWS Account | `692699826578` | Account |
| AWS Region | `us-east-1` | Region |

---

## Cleanup

To avoid ongoing charges, delete all resources in reverse order of creation:

```bash
# Scale down the service to zero tasks
aws ecs update-service \
  --cluster xfusion-cluster \
  --service xfusion-service \
  --desired-count 0 \
  --region us-east-1

# Delete the ECS service
aws ecs delete-service \
  --cluster xfusion-cluster \
  --service xfusion-service \
  --region us-east-1

# Deregister the task definition
aws ecs deregister-task-definition \
  --task-definition xfusion-taskdefinition:1 \
  --region us-east-1

# Delete the ECS cluster
aws ecs delete-cluster \
  --cluster xfusion-cluster \
  --region us-east-1

# Delete all images and the ECR repository
aws ecr batch-delete-image \
  --repository-name xfusion-ecr \
  --image-ids imageTag=latest \
  --region us-east-1

aws ecr delete-repository \
  --repository-name xfusion-ecr \
  --region us-east-1

# Delete the CloudWatch log group
aws logs delete-log-group \
  --log-group-name /ecs/xfusion-taskdefinition \
  --region us-east-1

# Detach policy and delete the IAM role
aws iam detach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam delete-role \
  --role-name ecsTaskExecutionRole

# Revoke the port 80 inbound rule
aws ec2 revoke-security-group-ingress \
  --group-id sg-06cc7634e44dc3fbe \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

---

## Troubleshooting

| Symptom | Root Cause | Resolution |
|---|---|---|
| Task stuck in `PENDING` | Execution role missing or misconfigured | Verify `ecsTaskExecutionRole` has `AmazonECSTaskExecutionRolePolicy` attached |
| Image pull failure | ECR login token expired | Re-run `aws ecr get-login-password` and `docker login` |
| Task exits immediately | Log group does not exist | Create `/ecs/xfusion-taskdefinition` log group before service creation |
| Port 80 unreachable | Security group missing inbound rule | Run `authorize-security-group-ingress` for TCP 80 |
| `curl` returns connection refused | `assignPublicIp=DISABLED` | Recreate service with `assignPublicIp=ENABLED` in `awsvpcConfiguration` |
| Service not reaching desired count | Subnet or AZ capacity issue | Try a different `SubnetId` in the same region |

---
