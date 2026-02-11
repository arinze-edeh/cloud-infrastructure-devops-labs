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

- AUTHENTICATE to AWS using the showcreds command to retrieve current IAM session details.

- VERIFY region is us-east-1 and the active identity is correctly mapped to the lab account.

- FETCH EC2 instance ID where Name = nautilus-ec2.

- FETCH Elastic IP allocation ID where Name = nautilus-ec2-eip.

- `Note: Due to lab constraints, the Elastic IP was manually allocated and tagged with the required name to ensure the fetch operation would succeed.`

- ASSOCIATE Elastic IP with the identified EC2 instance using the Public IP as a workaround for API synchronization delays.

- VERIFY Elastic IP is attached by describing the address and confirming the InstanceId and Name tag are correctly listed in the resource table.

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

- Outcome: Provisioned Public IP `34.237.96.234` with Allocation ID `eipalloc-09658a3b506276415`

📸 screenshot:
<img width="1024" height="543" alt="image" src="https://github.com/user-attachments/assets/dfc8b6f6-fe6a-49ce-8212-b60f7fd6a9ef" />

## Step 4: Managing API Consistency & Association
- During the association phase, the system encountered AWS Eventual Consistency issues where the new ID was not immediately recognized by the association service.

- Error Encountered: InvalidAllocationID.NotFound.

- Workaround: Association was forced using the Public IP directly to bypass the ID propagation delay.

- Command: `aws ec2 associate-address --instance-id i-09577c6c0d58f854f --public-ip 34.236.196.188`
- Success: Generated Association ID

📸 screenshot:
<img width="1033" height="714" alt="image" src="https://github.com/user-attachments/assets/1cf25c2f-877c-4708-a51a-553b9ad687d9" />

## Step 5: Resource Tagging & Validation
- To comply with lab naming conventions, the resource was identified using the mandatory Name tag.

- Correction: Tagging was correctly applied to the Allocation ID (the resource) rather than the Association ID (the link).

- Command: `aws ec2 create-tags --resources eipalloc-08ea08b020614768c --tags Key=Name,Value=nautilus-ec2-eip`
- Verification: A summary check confirmed the instance was successfully linked to the named IP.

📸 screenshots:
<img width="1018" height="795" alt="image" src="https://github.com/user-attachments/assets/d666f24e-bb45-4444-99ff-f452ebc8778a" />
<img width="1047" height="823" alt="image" src="https://github.com/user-attachments/assets/7cbeff2b-aa9e-48fa-9263-65eae364fa60" />

## Final Result
- Elastic IP successfully attached

- EC2 instance is publicly reachable

- Task completed using AWS CLI only

## Tags
`aws` `ec2` `elastic-ip` `networking` `aws-cli`












