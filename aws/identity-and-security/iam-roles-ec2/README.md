# AWS IAM Role for EC2 — Security-First Implementation

## Overview
- This lab demonstrates the creation of an AWS IAM Role for EC2 using the AWS CLI, following **least-privilege**, **explicit trust relationships**, and **credential verification** best practices.

- The role is configured with:
  -  A service trust relationship for EC2
  -  An existing customer-managed IAM policy
  -  Deployment scoped strictly to **us-east-1**

- This mirrors real-world production workflows used in regulated and large-scale cloud environments.

---

## Environment & Scope
| Item | Value |
|----|----|
| AWS Account | `472012609282` |
| Region | `us-east-1` |
| IAM Role | `iamrole_kareem` |
| Trusted Entity | `EC2` |
| Policy Attached | `iampolicy_kareem` |
| Access Type | `Temporary credentials` |

---

## Step 1: Identity Verification (Pre-flight Check)

- Before creating any IAM resources, the active AWS identity is validated to prevent cross-account or privilege misuse.

 - `aws sts get-caller-identity`

📸 Screenshot:
<img width="1034" height="535" alt="image" src="https://github.com/user-attachments/assets/69fcb71f-5cb3-4aa0-9d04-9a5e35e6486f" />


- Why this matters

  -  Prevents accidental deployment in the wrong AWS account

  -  Confirms non-root, auditable IAM usage

  -  Required in enterprise change-management workflows


## Step 2: Define Explicit Trust Policy (EC2 Only)

- A trust policy is created to explicitly allow only EC2 to assume the role.

cat <<EOF > trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

📸 Screenshot: screenshots/trust-policy-json.png

## Security Notes

- No wildcard principals

- No cross-account trust

- Service-scoped assumption only

## Step 3: Create IAM Role

- The IAM role is created using the defined trust policy.

aws iam create-role \
    --role-name iamrole_kareem \
    --assume-role-policy-document file://trust-policy.json

📸 Screenshot:

- Why roles (not users)

  -  Eliminates long-lived credentials

  -  Enables automatic credential rotation

  -  Required for EC2, ECS, and EKS workloads

## Step 4: Discover Policy ARN Dynamically

- The policy ARN is queried dynamically to avoid hard-coding and improve script portability.

POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='iampolicy_kareem'].Arn" \
  --output text)
echo $POLICY_ARN
arn:aws:iam::472012609282:policy/iampolicy_kareem

📸 Screenshot:

- Engineering Signal

- Automation-ready

- CI/CD friendly

- Eliminates environment-specific coupling

## Step 5: Attach Policy to Role

- The customer-managed policy is attached to the IAM role.

aws iam attach-role-policy \
    --role-name iamrole_kareem \
    --policy-arn $POLICY_ARN

📸 Screenshot:
<img width="1031" height="856" alt="image" src="https://github.com/user-attachments/assets/c4619b7b-bd8b-417f-a690-8abf6447a00b" />

## Step 6: Validate Role State

- Final validation confirms the policy attachment and role integrity.

aws iam list-attached-role-policies \
    --role-name iamrole_kareem
{
    "AttachedPolicies": [
        {
            "PolicyName": "iampolicy_kareem",
            "PolicyArn": "arn:aws:iam::472012609282:policy/iampolicy_kareem"
        }
    ]
}

📸 Screenshot:
<img width="1033" height="856" alt="image" src="https://github.com/user-attachments/assets/1d091c3a-05ff-4dee-acb0-ec347678e240" />

## Outcome

- IAM role created successfully
- EC2 configured as the only trusted service
- Policy attached and verified
- Deployment restricted to us-east-1

## Security & DevOps Principles Demonstrated

- Least-Privilege Access

- Explicit Trust Relationships

- Temporary Credential Usage

- Audit-Friendly CLI Operations

- Production-grade IAM design



<img width="1027" height="692" alt="image" src="https://github.com/user-attachments/assets/a84053b7-ee50-411e-9352-b40fac8d2a03" />
<img width="1019" height="860" alt="image" src="https://github.com/user-attachments/assets/63ae9ea4-a359-40d0-82e1-87c73f613bd6" />
<img width="1031" height="642" alt="image" src="https://github.com/user-attachments/assets/7697860a-21b4-4ac4-bf5f-6afb910bac68" />



