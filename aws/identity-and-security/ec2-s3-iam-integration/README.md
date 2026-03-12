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

---

### Phase 1: EC2 Instance Discovery and SSH Key Setup

---

#### Step 1.1 - Retrieve Instance Metadata

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

> ***Screenshot: Instance ID, Public IP, and AZ resolved successfully from the describe-instances query***

<img width="1033" height="606" alt="image" src="https://github.com/user-attachments/assets/732d4d03-1afb-4796-b9bc-46e7f477224f" />

---

#### Step 1.2 - Generate RSA SSH Key Pair

Generate a 4096-bit RSA key pair on the `aws-client` host. The private key remains local; the public key will be pushed to the instance in the next step.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
```

**Expected Output:**
```
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
```

> ***Screenshot: ssh-keygen output confirming both private and public key files written to /root/.ssh/ with the SHA256 fingerprint and randomart image***

<img width="1028" height="760" alt="image" src="https://github.com/user-attachments/assets/8ac83f81-ef40-4b0c-af47-2a925a164d47" />

---

#### Step 1.3 - Push Public Key via EC2 Instance Connect

Use EC2 Instance Connect to temporarily inject the public key into the instance for the `ubuntu` OS user. This is the secure, API-driven alternative to manual `authorized_keys` management.

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

> ***Screenshot: EC2 Instance Connect API response confirming Success: true and the associated RequestId***

<img width="1032" height="841" alt="image" src="https://github.com/user-attachments/assets/2d56ad1b-8b99-4ee8-bead-128e1483bb97" />

---

#### Step 1.4 - Authorize Root SSH Access

Bootstrap the `root` user's `authorized_keys` file by SSHing in as `ubuntu` and escalating with `sudo`. This enables direct root-level access for all subsequent automation steps.

```bash
PUB_KEY=$(cat ~/.ssh/id_rsa.pub)

ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa ubuntu@$EC2_PUBLIC_IP \
  "sudo mkdir -p /root/.ssh && \
  sudo chmod 700 /root/.ssh && \
  echo '$PUB_KEY' | sudo tee /root/.ssh/authorized_keys && \
  sudo chmod 600 /root/.ssh/authorized_keys"
```

> ***Screenshot: SSH session as ubuntu bootstrapping root authorized_keys, showing the public key written to /root/.ssh/authorized_keys via sudo tee***

![Step 1.4 - Root authorized_keys Bootstrap](screenshots/step-1.4-root-authorized-keys-bootstrap.png)

---

#### Step 1.4a - Verify Root SSH Access

```bash
ssh -i ~/.ssh/id_rsa root@$EC2_PUBLIC_IP "whoami"
# Expected: root
```

> ***Screenshot: whoami returning root, confirming successful root-level SSH access via the newly provisioned key pair***

![Step 1.4a - Root SSH Access Verified](screenshots/step-1.4a-root-ssh-whoami.png)

---

> ### Phase 1 Complete
> **EC2 instance discovered, SSH key pair generated, public key injected via EC2 Instance Connect, and root SSH access confirmed.**

> ***Screenshot: Full Phase 1 terminal session showing all steps from instance discovery through root SSH access verification in sequence***

![Phase 1 Complete - EC2 Discovery and SSH Setup](screenshots/phase-1-complete-overview.png)

---

### Phase 2: Private S3 Bucket Provisioning

---

#### Step 2.1 - Create the S3 Bucket

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

> ***Screenshot: s3api create-bucket response showing the Location field confirming bucket nautilus-s3-30029 was created in us-east-1***

![Step 2.1 - S3 Bucket Created](screenshots/step-2.1-s3-bucket-creation.png)

---

#### Step 2.2 - Block All Public Access

Enforce a full public access block on the bucket. All four flags must be `true` to achieve a fully hardened private bucket posture and prevent any ACL or bucket policy from inadvertently exposing contents publicly.

```bash
aws s3api put-public-access-block \
  --bucket nautilus-s3-30029 \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

> **Note:** A silent exit with no output and exit code 0 confirms the public access block was applied successfully.

> ***Screenshot: put-public-access-block command completing with zero error output, confirming all four block flags applied to nautilus-s3-30029***

![Step 2.2 - S3 Public Access Blocked](screenshots/step-2.2-s3-public-access-block.png)

---

> ### Phase 2 Complete
> **Private S3 bucket nautilus-s3-30029 provisioned with all public access blocked across ACLs and bucket policies.**

> ***Screenshot: Full Phase 2 terminal session showing bucket creation and public access block commands executed without errors***

![Phase 2 Complete - S3 Bucket Provisioned](screenshots/phase-2-complete-overview.png)

---

### Phase 3: IAM Policy and Role Configuration

---

#### Step 3.1 - Author the S3 IAM Policy Document

Scope the policy to the minimum required actions on the specific bucket ARN and its object prefix. Both ARN forms are required: one for bucket-level actions (`s3:ListBucket`) and one for object-level actions (`s3:PutObject`, `s3:GetObject`).

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

> ***Screenshot: cat heredoc writing nautilus-s3-policy.json to /tmp with the scoped Actions and dual Resource ARNs visible in the terminal***

![Step 3.1 - IAM Policy Document Authored](screenshots/step-3.1-iam-policy-document.png)

---

#### Step 3.2 - Create the IAM Policy

```bash
aws iam create-policy \
  --policy-name nautilus-s3-policy \
  --policy-document file:///tmp/nautilus-s3-policy.json
```

**Expected Output:**
```json
{
    "Policy": {
        "PolicyName": "nautilus-s3-policy",
        "PolicyId": "ANPAXLCLAV2YVI5ZATJTC",
        "Arn": "arn:aws:iam::504815988401:policy/nautilus-s3-policy",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "IsAttachable": true
    }
}
```

> ***Screenshot: IAM create-policy JSON response showing PolicyName, PolicyId, Arn, and AttachmentCount: 0 confirming the policy was created and is ready for attachment***

![Step 3.2 - IAM Policy Created](screenshots/step-3.2-iam-policy-created.png)

---

#### Step 3.3 - Resolve the Policy ARN

```bash
POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='nautilus-s3-policy'].Arn" \
  --output text)
```

> ***Screenshot: list-policies query returning the full ARN of nautilus-s3-policy and storing it into the POLICY_ARN shell variable***

![Step 3.3 - Policy ARN Resolved](screenshots/step-3.3-policy-arn-resolved.png)

---

#### Step 3.4 - Author the EC2 Trust Policy

Define the trust relationship document that permits the EC2 service principal to call `sts:AssumeRole` and receive temporary credentials scoped to this role.

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

> ***Screenshot: cat heredoc writing nautilus-trust.json to /tmp with the ec2.amazonaws.com principal and sts:AssumeRole action visible***

![Step 3.4 - EC2 Trust Policy Document Authored](screenshots/step-3.4-trust-policy-document.png)

---

#### Step 3.5 - Create the IAM Role

```bash
aws iam create-role \
  --role-name nautilus-role \
  --assume-role-policy-document file:///tmp/nautilus-trust.json
```

**Expected Output:**
```json
{
    "Role": {
        "RoleName": "nautilus-role",
        "RoleId": "AROAXLCLAV2Y7YZ5O7USL",
        "Arn": "arn:aws:iam::504815988401:role/nautilus-role",
        "CreateDate": "2026-03-12T06:23:31Z"
    }
}
```

> ***Screenshot: IAM create-role JSON response showing RoleName, RoleId, Arn, and CreateDate confirming nautilus-role was created with the EC2 trust relationship***

![Step 3.5 - IAM Role Created](screenshots/step-3.5-iam-role-created.png)

---

#### Step 3.6 - Attach the S3 Policy to the Role

```bash
aws iam attach-role-policy \
  --role-name nautilus-role \
  --policy-arn $POLICY_ARN
```

> **Note:** A silent exit with no output and exit code 0 confirms the policy was attached successfully.

> ***Screenshot: attach-role-policy completing with zero error output, confirming nautilus-s3-policy is now bound to nautilus-role***

![Step 3.6 - Policy Attached to Role](screenshots/step-3.6-policy-attached-to-role.png)

---

> ### Phase 3 Complete
> **IAM policy nautilus-s3-policy created with least-privilege S3 permissions. IAM role nautilus-role created with EC2 trust relationship. Policy successfully attached to the role.**

> ***Screenshot: Full Phase 3 terminal session showing policy document creation, IAM policy creation, role creation, and policy-role attachment in sequence***

![Phase 3 Complete - IAM Policy and Role Configured](screenshots/phase-3-complete-overview.png)

---

### Phase 4: Instance Profile Association

---

#### Step 4.1 - Create the Instance Profile

An instance profile is the container object that bridges an IAM role to an EC2 instance at the hypervisor level. The profile name matches the role name for operational consistency.

```bash
aws iam create-instance-profile \
  --instance-profile-name nautilus-role
```

**Expected Output:**
```json
{
    "InstanceProfile": {
        "InstanceProfileName": "nautilus-role",
        "InstanceProfileId": "AIPAXLCLAV2YZIE7AE57E",
        "Arn": "arn:aws:iam::504815988401:instance-profile/nautilus-role",
        "Roles": []
    }
}
```

> ***Screenshot: create-instance-profile JSON response showing the InstanceProfileName, Arn, and empty Roles array before the role is bound***

![Step 4.1 - Instance Profile Created](screenshots/step-4.1-instance-profile-created.png)

---

#### Step 4.2 - Add the Role to the Instance Profile

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name nautilus-role \
  --role-name nautilus-role
```

> ***Screenshot: add-role-to-instance-profile completing silently, confirming nautilus-role is bound inside the nautilus-role instance profile***

![Step 4.2 - Role Added to Instance Profile](screenshots/step-4.2-role-added-to-profile.png)

---

#### Step 4.3 - Associate the Instance Profile with the EC2 Instance

Allow 15 seconds for IAM propagation before associating to prevent eventual consistency race conditions in the control plane.

```bash
sleep 15

aws ec2 associate-iam-instance-profile \
  --instance-id $INSTANCE_ID \
  --iam-instance-profile Name=nautilus-role
```

**Expected Output:**
```json
{
    "IamInstanceProfileAssociation": {
        "AssociationId": "iip-assoc-0d1d4b1a6e7dbb9e4",
        "InstanceId": "i-0627c6806098d29f1",
        "IamInstanceProfile": {
            "Arn": "arn:aws:iam::504815988401:instance-profile/nautilus-role"
        },
        "State": "associating"
    }
}
```

> ***Screenshot: associate-iam-instance-profile JSON response showing AssociationId, InstanceId, and initial State: associating***

![Step 4.3 - Instance Profile Association Initiated](screenshots/step-4.3-instance-profile-associating.png)

---

#### Step 4.4 - Confirm Association

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

> ***Screenshot: describe-instances table output confirming both the instance ID and the nautilus-role instance profile ARN are present and fully associated***

![Step 4.4 - Instance Profile Association Confirmed](screenshots/step-4.4-instance-profile-confirmed.png)

---

> ### Phase 4 Complete
> **Instance profile nautilus-role created, role bound to profile, and profile successfully associated with the nautilus-ec2 instance. The instance now retrieves temporary credentials from IMDS automatically on every AWS API call.**

> ***Screenshot: Full Phase 4 terminal session showing instance profile creation, role binding, association initiation, and describe-instances table confirmation***

![Phase 4 Complete - Instance Profile Associated](screenshots/phase-4-complete-overview.png)

---

### Phase 5: Access Verification

---

#### Step 5.1 - SSH into the EC2 Instance

```bash
ssh -i ~/.ssh/id_rsa root@$EC2_PUBLIC_IP
```

> ***Screenshot: Successful SSH login to the EC2 instance as root showing the Ubuntu 22.04 LTS welcome banner, system uptime, memory usage, and IP address***

![Step 5.1 - SSH Into EC2 Instance](screenshots/step-5.1-ssh-into-ec2.png)

---

#### Step 5.2 - Create and Upload a Test File to S3

From inside the instance, create a test file and upload it to the bucket. Authentication occurs automatically via the attached instance profile with zero static credentials required anywhere on disk.

```bash
echo "Nautilus DevOps Test File" > testfile.txt
aws s3 cp testfile.txt s3://nautilus-s3-30029/
```

**Expected Output:**
```
upload: ./testfile.txt to s3://nautilus-s3-30029/testfile.txt
```

> ***Screenshot: Inside the EC2 instance showing the echo command creating testfile.txt and the aws s3 cp command confirming a successful credential-free upload to nautilus-s3-30029***

![Step 5.2 - Test File Uploaded to S3](screenshots/step-5.2-s3-upload.png)

---

#### Step 5.3 - List Bucket Contents

```bash
aws s3 ls s3://nautilus-s3-30029/
```

**Expected Output:**
```
2026-03-12 06:28:26    26 testfile.txt
```

A successful listing confirms the EC2 instance is authenticating to S3 via the IAM role with zero static credentials on disk.

> ***Screenshot: aws s3 ls output inside the EC2 instance showing testfile.txt with its upload timestamp and size, confirming end-to-end read access via the instance profile***

![Step 5.3 - S3 Bucket Contents Listed](screenshots/step-5.3-s3-ls-output.png)

---

> ### Phase 5 Complete
> **End-to-end access verified. The EC2 instance successfully uploaded and listed objects in nautilus-s3-30029 using IAM role credentials retrieved automatically from the instance metadata service. No static credentials were used at any point in this workflow.**

> ***Screenshot: Full Phase 5 terminal session showing SSH login, testfile.txt creation, s3 cp upload, and s3 ls listing inside the EC2 instance confirming complete success***

![Phase 5 Complete - S3 Access Verified](screenshots/phase-5-complete-overview.png)

---

## Security Considerations

| Risk | Mitigation Applied |
|---|---|
| Static AWS credentials on EC2 | Eliminated via IAM Instance Profile (IMDS-based auth) |
| Overly permissive S3 access | Policy scoped to `PutObject`, `GetObject`, `ListBucket` only |
| Public S3 bucket exposure | All four public access block flags enforced |
| Broad SSH access | Key-based auth only; `StrictHostKeyChecking` bypassed only for initial bootstrap |
| IAM role over-permission | Trust policy restricted to `ec2.amazonaws.com` service principal only |

> **Important:** The `sleep 15` before instance profile association accounts for IAM eventual consistency. In automated pipelines, replace this with a polling loop that checks the association `State` field before proceeding.

---

## Troubleshooting

**Issue:** `aws s3 cp` fails inside the EC2 instance with `Unable to locate credentials`

**Resolution:** Verify the instance profile is fully associated and confirm IMDS is reachable:
```bash
aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].IamInstanceProfile"

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```
If `State` is still `associating`, wait and retry. If IMDS is unreachable, verify the instance network configuration permits traffic to `169.254.169.254`.

---

**Issue:** SSH connection refused after pushing public key via EC2 Instance Connect

**Resolution:** EC2 Instance Connect keys expire after 60 seconds. Re-run the `send-ssh-public-key` command immediately before connecting, or verify the root `authorized_keys` bootstrap step completed successfully within the same session window.

---

**Issue:** `AccessDenied` error when calling `s3:ListBucket`

**Resolution:** Confirm both ARN entries exist in the policy resource block. `s3:ListBucket` requires the bucket ARN (`arn:aws:s3:::nautilus-s3-30029`) while object-level actions require the object prefix ARN (`arn:aws:s3:::nautilus-s3-30029/*`). Both must be present in the same statement.

---

## References

- [AWS EC2 Instance Profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
- [AWS EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Connect-using-EC2-Instance-Connect.html)
- [S3 Block Public Access](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [IAM Least Privilege Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS CLI S3 Commands](https://docs.aws.amazon.com/cli/latest/reference/s3/)

---

*Maintained by the Nautilus DevOps Team | Infrastructure and Platform Engineering*




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
