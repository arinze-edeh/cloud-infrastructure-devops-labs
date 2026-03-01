# Private Amazon ECR Repository Creation & Docker Image Deployment

---

## Problem Statement

The Nautilus DevOps team requires a **secure, centralized, and scalable container image registry** to support modern application delivery workflows. The objective is to:

- Provision a **private container registry**
- Build a Docker image from an existing application source
- Securely authenticate and push the image to the registry
- Validate successful image availability for downstream deployments

This task must be executed strictly within the **us-east-1** region using approved AWS credentials, following enterprise-grade DevOps best practices.

---

## Objective

- Create a **private Amazon Elastic Container Registry (ECR)** repository
- Build a Docker image from a provided Dockerfile
- Tag the image as `latest`
- Push the image to the newly created ECR repository
- Verify the image exists in the repository

---

## Architecture Overview

```
Developer Host (aws-client)
        |
        |  Docker Build
        v
Local Docker Image (latest)
        |
        |  Authenticated Push
        v
Private ECR Repository (us-east-1)
```

## Environment & Constraints
| Parameter           | Value                             |
| ------------------- | --------------------------------- |
| Cloud Provider      | Amazon Web Services               |
| Container Registry  | Amazon Elastic Container Registry |
| Region              | us-east-1                         |
| Image Tag           | `latest`                          |
| Repository Name     | `nautilus-ecr`                    |
| Dockerfile Location | `/root/pyapp`                     |
| Access Type         | Private Repository                |

## Solution Strategy

- Validate AWS identity and permissions

- Create a private ECR repository

- Build the Docker image locally

- Authenticate Docker with ECR

- Tag and push the image

- Verify successful image upload

## Step-by-Step Implementation

## Step 1: Verify AWS Identity
```
aws sts get-caller-identity
```
Expected Outcome

- Valid AWS Account ID

- IAM user identity confirmed

📸 Screenshot Placeholder
#

## Step 2: Create Private ECR Repository
```
aws ecr create-repository \
  --repository-name nautilus-ecr \
  --region us-east-1
```
Key Outputs

- Repository ARN

- Repository URI

- Encryption enabled (AES256)

📸 Screenshot:


## Step 3: avigate to Application Directory
```
cd /root/pyapp
```
📸 Screenshot:


## Step 4: Build Docker Image from Dockerfile
```
docker build -t nautilus-ecr:latest .
```
Validation

- Build completes successfully

- Image tagged as latest

📸 Screenshot:


## Step 5: Authenticate Docker with ECR
```
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin 821328497772.dkr.ecr.us-east-1.amazonaws.com
```

Security Note

- Authentication token is time-bound

- No plaintext credentials transmitted

📸 Screenshot:


## Step 6: Tag Image for ECR
```
docker tag nautilus-ecr:latest \
821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr:latest
```
📸 Screenshot:


7️⃣ Push Image to ECR
docker push 821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr:latest

Expected Result

All image layers pushed successfully

Digest generated for integrity verification

8️⃣ Verify Image Availability
aws ecr list-images --repository-name nautilus-ecr

Expected Output

latest tag present

Image digest visible

✅ Final Validation Checklist
Check	Status
ECR Repository Created	✔
Docker Image Built	✔
Image Tagged Correctly	✔
Image Pushed to ECR	✔
Image Verified in ECR	✔
Region Compliance	✔ us-east-1
📦 Deliverables

Private ECR Repository: nautilus-ecr

Docker Image Tag: latest

Registry URI:
821328497772.dkr.ecr.us-east-1.amazonaws.com/nautilus-ecr

🧩 Key Learnings

ECR provides secure, IAM-integrated container storage

Docker image tagging must align with ECR repository URIs

Authentication via get-login-password is the recommended modern approach

Image verification ensures deployment readiness

🚀 Next Steps (Optional Enhancements)

Enable image scanning on push

Add lifecycle policies for image retention

Integrate with CI/CD pipelines

Apply immutable image tags for production workloads

🏁 Conclusion

This implementation successfully establishes a production-ready container image workflow using AWS-native services. The Docker image is securely built, authenticated, pushed, and verified in a private ECR repository, aligning with enterprise DevOps standards and cloud security best practices.

<img width="1032" height="626" alt="image" src="https://github.com/user-attachments/assets/67326ba6-d431-4512-b651-9455990240d6" />
<img width="1035" height="723" alt="image" src="https://github.com/user-attachments/assets/cc7d219d-3ffc-4944-9f85-984b7ec2f4ea" />
<img width="1026" height="701" alt="image" src="https://github.com/user-attachments/assets/52738a9f-cff0-4680-930e-53c25aa375c4" />
<img width="1039" height="664" alt="image" src="https://github.com/user-attachments/assets/c8f0105e-d172-4eab-a1e0-4d571c1a2e61" />
<img width="1038" height="785" alt="image" src="https://github.com/user-attachments/assets/bee87ed7-4175-4565-be90-fa4d3048211f" />
<img width="1040" height="837" alt="image" src="https://github.com/user-attachments/assets/2cb46a28-ce5d-41f5-8d11-f12b907d0135" />
<img width="1037" height="471" alt="image" src="https://github.com/user-attachments/assets/a750cd1d-723b-4e3a-8637-eb82fa0688d6" />
<img width="1031" height="665" alt="image" src="https://github.com/user-attachments/assets/54d2759f-9b7a-410d-a6e1-0c6f3e5bcc44" />

