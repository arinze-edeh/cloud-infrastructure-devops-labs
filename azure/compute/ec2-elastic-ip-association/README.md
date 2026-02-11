# Attach Elastic IP to EC2 Instance (AWS)

## Lab Overview
This lab demonstrates how to attach an existing Elastic IP address
to an existing EC2 instance using the AWS CLI in the us-east-1 region.

---

## Objective
- Identify an EC2 instance
- Identify an Elastic IP
- Attach the Elastic IP to the EC2 instance
- Verify successful association

---

## High-Level Flow (Pseudo-Code)

AUTHENTICATE to AWS
VERIFY region is us-east-1

FETCH EC2 instance ID where Name = xfusion-ec2
FETCH Elastic IP allocation ID where name = xfusion-ec2-eip

ASSOCIATE Elastic IP with EC2 instance
VERIFY Elastic IP is attached

## Implementation Steps
Step 1: Retrieve AWS Credentials
- The local terminal environment was synchronized with the provided IAM credentials to establish a secure management session.

- Action: Execute `showcreds` to retrieve temporary security tokens.

- Verification: Run aws sts get-caller-identity to confirm the active user is kk_labs_user_164676.

- Region Config: Validated that the CLI was targeting us-east-1 via aws configure get region.

📸 screenshot:
<img width="1036" height="654" alt="image" src="https://github.com/user-attachments/assets/a23c1cdc-fe91-462e-ad00-02e285a87d37" />
<img width="1032" height="577" alt="image" src="https://github.com/user-attachments/assets/383aa7f6-606e-4813-82a3-2c7e678d6c6d" />

## Step 2: Resource Discovery
- Target infrastructure was audited to identify the specific EC2 instance requiring a static public endpoint.

- Action: List instances with a filter for the name nautilus-ec2.

- Command:
`aws ec2 describe-instances --filters "Name=tag:Name,Values=nautilus-ec2" --query "Reservations[].Instances[].InstanceId" --output text`
- Result: Identified Target Instance ID

📸 screenshot:
<img width="1002" height="557" alt="image" src="https://github.com/user-attachments/assets/c60c15a2-e25b-4796-a06c-f589a6c1c67d" />

## Step 3: Elastic IP Allocation
- A new static IP address was provisioned from the Amazon pool to provide a persistent entry point.

- Action: Allocate a VPC-domain Elastic IP address.

- Command: `aws ec2 allocate-address --domain vpc --region us-east-1`

- Outcome: Provisioned Public IP `34.236.196.188` with Allocation ID `eipalloc-08ea08b020614768c`

📸 screenshot:
<img width="1024" height="543" alt="image" src="https://github.com/user-attachments/assets/dfc8b6f6-fe6a-49ce-8212-b60f7fd6a9ef" />

## Step 4: Get Elastic IP Allocation ID
aws ec2 describe-addresses \
  --query "Addresses[].{PublicIp:PublicIp,AllocationId:AllocationId}" \
  --output table
📸 screenshots/eip-allocation-id.png

## Step 5: Attach Elastic IP
aws ec2 associate-address \
  --instance-id <INSTANCE_ID> \
  --allocation-id <ALLOCATION_ID>
📸 screenshots/eip-associated.png

## Step 6: Verify Attachment
aws ec2 describe-addresses \
  --allocation-ids <ALLOCATION_ID>
📸 screenshots/eip-verification.png

## Final Result
Elastic IP successfully attached

EC2 instance is publicly reachable

Task completed using AWS CLI only

🏷️ Tags
`aws` `ec2` `elastic-ip` `networking` `aws-cli`






<img width="1041" height="661" alt="image" src="https://github.com/user-attachments/assets/d75afb54-7e95-440e-b237-0340a8cf3d4b" />
<img width="1033" height="714" alt="image" src="https://github.com/user-attachments/assets/1cf25c2f-877c-4708-a51a-553b9ad687d9" />
<img width="1018" height="795" alt="image" src="https://github.com/user-attachments/assets/d666f24e-bb45-4444-99ff-f452ebc8778a" />
<img width="1047" height="823" alt="image" src="https://github.com/user-attachments/assets/7cbeff2b-aa9e-48fa-9263-65eae364fa60" />



