# EC2 to S3 Integration via IAM Role and Instance Profile

> **Enterprise DevOps Infrastructure Setup** | AWS | IAM | EC2 | S3 | Ubuntu 22.04

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Phase 1: EC2 Instance Discovery and SSH Key Setup](#phase-1-ec2-instance-discovery-and-ssh-key-setup)
  - [Phase 2: Private S3 Bucket Provisioning](#phase-2-private-s3-bucket-provisioning)
  - [Phase 3: IAM Policy and Role Configuration](#phase-3-iam-policy-and-role-configuration)
  - [Phase 4: Instance Profile Association](#phase-4-instance-profile-association)
  - [Phase 5: Access Verification](#phase-5-access-verification)
- [Screenshots](#screenshots)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Problem Statement

A production EC2 instance (`nautilus-ec2`) required programmatic, credential-free access to a private S3 bucket for storing and retrieving application data. Hardcoding AWS credentials inside an EC2 instance is a critical security anti-pattern. The resolution required a fully IAM-driven approach using an instance profile attached to a scoped IAM role, following the AWS principle of least privilege.

**Key Challenges:**

- Establishing secure SSH access to the EC2 instance from the `aws-client` host without pre-existing key material
- Provisioning a private S3 bucket with all public access blocked
- Granting the EC2 instance scoped S3 permissions without static credentials
- Validating end-to-end bucket access directly from within the instance

---

## Solution Overview

The resolution implements AWS EC2 Instance Profile-based authentication, which allows the EC2 instance to assume an IAM role automatically via the instance metadata service (IMDS). This eliminates the need for long-lived access keys on the instance and enables auditable, revocable access control.

**Core Components:**

- **EC2 Instance:** `nautilus-ec2` (running, `us-east-1a`)
- **S3 Bucket:** `nautilus-s3-30029` (private, public access fully blocked)
- **IAM Policy:** `nautilus-s3-policy` (scoped to `PutObject`, `GetObject`, `ListBucket`)
- **IAM Role:** `nautilus-role` (EC2 trust relationship, policy attached)
- **Instance Profile:** `nautilus-role` (associated to EC2 instance)

---

## Architecture

```
+------------------+        AssumeRole (IMDS)        +----------------------+
|   nautilus-ec2   | --------------------------------> |    nautilus-role      |
|  (us-east-1a)    |                                  |  (IAM Role / EC2 TP) |
+------------------+                                  +----------+-----------+
         |                                                       |
         | aws s3 cp / ls                                        | nautilus-s3-policy
         |                                            +----------+-----------+
         +------------------------------------------> |   nautilus-s3-30029  |
                                                      |   (Private S3 Bucket)|
                                                      +----------------------+
```

**Data Flow:**
1. EC2 instance requests temporary credentials from the IMDS endpoint via the attached instance profile
2. AWS STS issues short-lived credentials scoped to `nautilus-role`
3. The instance uses those credentials to authenticate S3 API calls
4. S3 enforces the `nautilus-s3-policy` boundary on every request

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS CLI | Configured with permissions to manage EC2, S3, and IAM |
| Region | `us-east-1` |
| Existing EC2 Instance | Tagged `Name=nautilus-ec2`, state `running` |
| `ssh-keygen` | Available on the `aws-client` host |
| EC2 Instance Connect | Enabled on the target instance |

---

## Implementation

### Phase 1: EC2 Instance Discovery and SSH Key Setup

**1.1 Retrieve Instance Metadata**

Query the running instance by tag name and extract the instance ID, public IP, and availability zone for use in subsequent commands.

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)

EC2_PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].PublicIpAddress" \
  --output text)

AZ=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].Placement.AvailabilityZone" \
  --output text)

echo "Instance: $INSTANCE_ID | IP: $EC2_PUBLIC_IP | AZ: $AZ"
```

**Expected Output:**
```
Instance: i-0627c6806098d29f1 | IP: 54.146.248.93 | AZ: us-east-1a
```

---

**1.2 Generate RSA SSH Key Pair**

Generate a 4096-bit RSA key pair on the `aws-client` host. The private key stays local; the public key is pushed to the instance.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

---

**1.3 Push Public Key via EC2 Instance Connect**

Use EC2 Instance Connect to temporarily inject the public key into the instance for the `ubuntu` OS user. This is the secure, API-driven alternative to manual authorized_keys management.

```bash
aws ec2-instance-connect send-ssh-public-key \
  --instance-id $INSTANCE_ID \
  --instance-os-user ubuntu \
  --ssh-public-key file://~/.ssh/id_rsa.pub \
  --availability-zone $AZ
```

**Expected Output:**
```json
{
    "RequestId": "d2832136-c3b1-41d5-a7ce-cead264a2465",
    "Success": true
}
```

---

**1.4 Authorize Root SSH Access**

Bootstrap the `root` user's `authorized_keys` file by SSHing as `ubuntu` and escalating with `sudo`. This enables direct root-level access for subsequent automation steps.

```bash
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)

ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ubuntu@$EC2_PUBLIC_IP \
  "sudo mkdir -p /root/.ssh && \
  sudo chmod 700 /root/.ssh && \
  echo '$PUB_KEY' | sudo tee /root/.ssh/authorized_keys && \
  sudo chmod 600 /root/.ssh/authorized_keys"
```

**Verify root access:**
```bash
ssh -i ~/.ssh/id_rsa root@$EC2_PUBLIC_IP "whoami"
# Expected: root
```

---

### Phase 2: Private S3 Bucket Provisioning

**2.1 Create the S3 Bucket**

```bash
aws s3api create-bucket \
  --bucket nautilus-s3-30029 \
  --region us-east-1
```

**Expected Output:**
```json
{
    "Location": "/nautilus-s3-30029"
}
```

---

**2.2 Block All Public Access**

Enforce a full public access block on the bucket. This prevents any ACL or policy from inadvertently exposing bucket contents publicly.

```bash
aws s3api put-public-access-block \
  --bucket nautilus-s3-30029 \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> **Note:** All four flags must be set to `true` to achieve a fully hardened private bucket posture.

---

### Phase 3: IAM Policy and Role Configuration

**3.1 Author the S3 IAM Policy Document**

Scope the policy to the minimum required actions (`PutObject`, `GetObject`, `ListBucket`) on the specific bucket ARN and its object prefix.

```bash
cat > /tmp/nautilus-s3-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::nautilus-s3-30029",
        "arn:aws:s3:::nautilus-s3-30029/*"
      ]
    }
  ]
}
EOF
```

---

**3.2 Create the IAM Policy**

```bash
aws iam create-policy \
  --policy-name nautilus-s3-policy \
  --policy-document file:///tmp/nautilus-s3-policy.json
```

**Expected Output (truncated):**
```json
{
    "Policy": {
        "PolicyName": "nautilus-s3-policy",
        "Arn": "arn:aws:iam::504815988401:policy/nautilus-s3-policy",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true
    }
}
```

---

**3.3 Resolve the Policy ARN**

```bash
POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='nautilus-s3-policy'].Arn" \
  --output text)
```

---

**3.4 Author the EC2 Trust Policy**

Define the trust relationship document that allows the EC2 service principal to assume the role.

```bash
cat > /tmp/nautilus-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF
```

---

**3.5 Create the IAM Role**

```bash
aws iam create-role \
  --role-name nautilus-role \
  --assume-role-policy-document file:///tmp/nautilus-trust.json
```

---

**3.6 Attach the S3 Policy to the Role**

```bash
aws iam attach-role-policy \
  --role-name nautilus-role \
  --policy-arn $POLICY_ARN
```

---

### Phase 4: Instance Profile Association

**4.1 Create the Instance Profile**

An instance profile is the container object that bridges an IAM role to an EC2 instance at the hypervisor level.

```bash
aws iam create-instance-profile \
  --instance-profile-name nautilus-role
```

---

**4.2 Add the Role to the Instance Profile**

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name nautilus-role \
  --role-name nautilus-role
```

---

**4.3 Associate the Instance Profile with the EC2 Instance**

Allow 15 seconds for IAM propagation before associating, to prevent eventual consistency errors.

```bash
sleep 15

aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=nautilus-role
```

---

**4.4 Confirm Association**

```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].[InstanceId,IamInstanceProfile.Arn]" \
  --output table
```

**Expected Output:**
```
--------------------------------------------------------------
|                      DescribeInstances                     |
+------------------------------------------------------------+
|  i-0627c6806098d29f1                                       |
|  arn:aws:iam::504815988401:instance-profile/nautilus-role  |
+------------------------------------------------------------+
```

---

### Phase 5: Access Verification

**5.1 SSH into the EC2 Instance**

```bash
ssh -i ~/.ssh/id_rsa root@$EC2_PUBLIC_IP
```

---

**5.2 Upload a Test File to S3**

From inside the instance, create a test file and upload it to the bucket. The upload will authenticate automatically via the attached instance profile with no credentials required.

```bash
echo "Nautilus DevOps Test File" > testfile.txt
aws s3 cp testfile.txt s3://nautilus-s3-30029/
```

**Expected Output:**
```
upload: ./testfile.txt to s3://nautilus-s3-30029/testfile.txt
```

---

**5.3 List Bucket Contents**

```bash
aws s3 ls s3://nautilus-s3-30029/
```

**Expected Output:**
```
2026-03-12 06:28:26    26 testfile.txt
```

A successful listing confirms that the EC2 instance is authenticating to S3 via its IAM role with zero static credentials.

---

## Screenshots

### EC2 Instance Metadata Resolution
> *Screenshot: Terminal output showing Instance ID, Public IP, and Availability Zone after running the describe-instances query*

```
[ INSERT SCREENSHOT: ec2-instance-metadata.png ]
```

---

### SSH Key Generation and EC2 Instance Connect
> *Screenshot: ssh-keygen output and successful EC2 Instance Connect API response showing `"Success": true`*

```
[ INSERT SCREENSHOT: ssh-key-ec2-connect.png ]
```

---

### Root SSH Access Confirmation
> *Screenshot: Terminal output confirming `whoami` returns `root` after connecting via the new key pair*

```
[ INSERT SCREENSHOT: root-ssh-access.png ]
```

---

### S3 Bucket Creation and Public Access Block
> *Screenshot: s3api create-bucket response and put-public-access-block command execution*

```
[ INSERT SCREENSHOT: s3-bucket-creation.png ]
```

---

### IAM Policy and Role Creation
> *Screenshot: IAM policy JSON output showing PolicyName, Arn, and AttachmentCount = 0 before attachment*

```
[ INSERT SCREENSHOT: iam-policy-role-creation.png ]
```

---

### Instance Profile Association
> *Screenshot: ec2 describe-instances table output confirming the nautilus-role instance profile ARN is attached to the instance*

```
[ INSERT SCREENSHOT: instance-profile-association.png ]
```

---

### S3 Upload and Listing from EC2
> *Screenshot: Inside the EC2 instance showing successful `aws s3 cp` upload and `aws s3 ls` listing of testfile.txt*

```
[ INSERT SCREENSHOT: s3-upload-verification.png ]
```

---

## Security Considerations

| Risk | Mitigation Applied |
|---|---|
| Static AWS credentials on EC2 | Eliminated via IAM Instance Profile (IMDS-based auth) |
| Overly permissive S3 access | Policy scoped to `PutObject`, `GetObject`, `ListBucket` only |
| Public S3 bucket exposure | All four public access block flags enforced |
| Broad SSH access | Key-based auth only; `StrictHostKeyChecking` bypassed only for initial bootstrap |
| IAM role over-permission | Trust policy restricted to `ec2.amazonaws.com` service principal only |

> **Important:** The `sleep 15` before instance profile association accounts for IAM eventual consistency. In automated pipelines, replace this with a polling loop that checks association state before proceeding.

---

## Troubleshooting

**Issue:** `aws s3 cp` fails inside the EC2 instance with `Unable to locate credentials`

**Resolution:** Verify the instance profile is fully associated:
```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].IamInstanceProfile"
```
If `State` is still `associating`, wait and retry. Also confirm IMDS is reachable from the instance:
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

**Issue:** SSH connection refused after pushing public key via EC2 Instance Connect

**Resolution:** EC2 Instance Connect keys expire after 60 seconds. Re-run the `send-ssh-public-key` command immediately before connecting, or ensure the root `authorized_keys` bootstrap step completed successfully in the same session window.

---

**Issue:** `AccessDenied` error when calling `s3:ListBucket`

**Resolution:** Confirm both ARN entries exist in the policy resource block. `s3:ListBucket` requires the bucket ARN (`arn:aws:s3:::nautilus-s3-30029`) while object-level actions require the object prefix ARN (`arn:aws:s3:::nautilus-s3-30029/*`). Both must be present.

---

## References

- [AWS EC2 Instance Profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
- [AWS EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)
- [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [IAM Least Privilege Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS CLI S3 Commands](https://docs.aws.amazon.com/cli/latest/reference/s3/)

---

*Maintained by the Nautilus DevOps Team | Infrastructure and Platform Engineering*

<img width="1033" height="606" alt="image" src="https://github.com/user-attachments/assets/732d4d03-1afb-4796-b9bc-46e7f477224f" />
<img width="1028" height="760" alt="image" src="https://github.com/user-attachments/assets/8ac83f81-ef40-4b0c-af47-2a925a164d47" />
<img width="1032" height="841" alt="image" src="https://github.com/user-attachments/assets/2d56ad1b-8b99-4ee8-bead-128e1483bb97" />
<img width="1031" height="528" alt="image" src="https://github.com/user-attachments/assets/fcde31ef-f4ea-44d5-a30a-294cbdfdad68" />
<img width="1035" height="799" alt="image" src="https://github.com/user-attachments/assets/6fcfc934-7058-4d5c-a99b-b8d68f2b52ce" />
<img width="1033" height="860" alt="image" src="https://github.com/user-attachments/assets/a3313f11-e173-4ccc-85f4-165f1be17d7a" />
<img width="1031" height="712" alt="image" src="https://github.com/user-attachments/assets/5495f695-e68c-4642-98d0-a6cb5ffd91ed" />
<img width="1030" height="836" alt="image" src="https://github.com/user-attachments/assets/65a1df8e-97c3-4196-89b0-b2746aefd231" />
<img width="1036" height="842" alt="image" src="https://github.com/user-attachments/assets/770715d2-e0df-4c37-b6ac-5653e260b457" />
<img width="1023" height="767" alt="image" src="https://github.com/user-attachments/assets/f6caff0c-0c37-4ca6-985c-92ce7c6889be" />
<img width="1030" height="658" alt="image" src="https://github.com/user-attachments/assets/75852bb6-2694-4648-bf4f-1d8ecfabd884" />
<img width="1023" height="783" alt="image" src="https://github.com/user-attachments/assets/71f6127d-6742-4905-af3e-816b003aac74" />
<img width="1026" height="859" alt="image" src="https://github.com/user-attachments/assets/9e4ae383-00c9-4c47-976e-1e82d0afa1ef" />
