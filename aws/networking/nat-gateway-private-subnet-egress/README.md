# AWS NAT Gateway Setup for Private Subnet Internet Access

![AWS](https://img.shields.io/badge/AWS-NAT_Gateway-orange?style=for-the-badge&logo=amazon-aws)
![VPC](https://img.shields.io/badge/VPC-Networking-blue?style=for-the-badge&logo=amazon-aws)
![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=for-the-badge)
![IaC](https://img.shields.io/badge/IaC-AWS_CLI-yellow?style=for-the-badge&logo=amazon-aws)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Solution Overview](#solution-overview)
- [Architecture Diagram](#architecture-diagram)
- [Prerequisites](#prerequisites)
- [Infrastructure Summary](#infrastructure-summary)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Discover Existing VPC Resources](#step-1-discover-existing-vpc-resources)
  - [Step 2: Create a Public Subnet](#step-2-create-a-public-subnet)
  - [Step 3: Create and Attach an Internet Gateway](#step-3-create-and-attach-an-internet-gateway)
  - [Step 4: Create a Public Route Table](#step-4-create-a-public-route-table)
  - [Step 5: Allocate an Elastic IP and Create the NAT Gateway](#step-5-allocate-an-elastic-ip-and-create-the-nat-gateway)
  - [Step 6: Create a Private Route Table and Route via NAT Gateway](#step-6-create-a-private-route-table-and-route-via-nat-gateway)
  - [Step 7: Verify End-to-End Connectivity](#step-7-verify-end-to-end-connectivity)
- [Verification and Validation](#verification-and-validation)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Resource Reference](#resource-reference)

---

## Problem Statement

The Nautilus DevOps team operates an EC2 instance (`xfusion-priv-ec2`) inside a **private subnet** with no direct internet access. The instance requires outbound internet connectivity to upload test artifacts to an S3 bucket (`xfusion-nat-3445`) via a scheduled cron job.

**Constraints:**

* The EC2 instance must remain in the private subnet (no public IP assignment).
* Outbound traffic must be routed securely through a managed NAT Gateway.
* The existing VPC (`xfusion-priv-vpc`) and private subnet (`xfusion-priv-subnet`) are pre-provisioned and must not be modified destructively.
* All resources must be tagged consistently for operational visibility.

---

## Solution Overview

This runbook documents the implementation of a **NAT Gateway architecture** that provides controlled, outbound-only internet access from a private subnet. The solution involves:

1. Creating a dedicated **public subnet** within the existing VPC.
2. Provisioning an **Internet Gateway** and attaching it to the VPC.
3. Configuring a **public route table** with a default route to the Internet Gateway.
4. Allocating an **Elastic IP** and deploying a **NAT Gateway** in the public subnet.
5. Updating the **private route table** to route all egress traffic through the NAT Gateway.

---

## Architecture Diagram

```
                          +------------------------------+
                          |     xfusion-priv-vpc         |
                          |     CIDR: 10.1.0.0/16        |
                          |                              |
  Internet                |  +------------------------+  |
     |                    |  |  Public Subnet         |  |
     |  <-- IGW -->       |  |  10.1.2.0/24           |  |
     |  igw-066f3fb1...   |  |  (xfusion-pub-subnet)  |  |
     |                    |  |                        |  |
     |                    |  |  [NAT Gateway]         |  |
     |                    |  |  xfusion-natgw         |  |
     |                    |  |  EIP: 32.193.164.228   |  |
     |                    |  +------------------------+  |
     |                    |            |                 |
     |                    |            | (Outbound Only) |
     |                    |  +------------------------+  |
     |                    |  |  Private Subnet        |  |
     |                    |  |  10.1.1.0/24           |  |
     |                    |  |  (xfusion-priv-subnet) |  |
     |                    |  |                        |  |
     |                    |  |  [xfusion-priv-ec2]    |  |
     |                    |  +------------------------+  |
     |                    +------------------------------+
     |
  [S3 Bucket: xfusion-nat-3445]
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS CLI | v2.x configured with appropriate IAM credentials |
| IAM Permissions | `ec2:*`, `s3:GetObject`, `sts:GetCallerIdentity` |
| Existing VPC | `xfusion-priv-vpc` tagged and available |
| Existing Private Subnet | `xfusion-priv-subnet` tagged and available |
| EC2 Instance | `xfusion-priv-ec2` running in the private subnet with a cron job configured |
| Region | `us-east-1` |

Verify your identity before proceeding:

```bash
aws sts get-caller-identity
```

**Expected output:**

```json
{
    "UserId": "AIDARL6WMRVVHSSHV7UWD",
    "Account": "094402022762",
    "Arn": "arn:aws:iam::094402022762:user/kk_labs_user_683011"
}
```

---

## Infrastructure Summary

| Resource | Name | ID | CIDR / Value |
|---|---|---|---|
| VPC | xfusion-priv-vpc | vpc-0cf7a9d60553403ab | 10.1.0.0/16 |
| Private Subnet | xfusion-priv-subnet | subnet-0f76f16569c103e13 | 10.1.1.0/24 |
| Public Subnet | xfusion-pub-subnet | subnet-0779d195127e0e801 | 10.1.2.0/24 |
| Internet Gateway | xfusion-igw | igw-066f3fb13549eba8e | N/A |
| Public Route Table | xfusion-pub-rt | rtb-0be5f139da73d23d9 | 0.0.0.0/0 via IGW |
| Private Route Table | xfusion-priv-rt | rtb-0887b242a83ac7110 | 0.0.0.0/0 via NAT |
| Elastic IP | N/A | eipalloc-08e5e560f682cfe6a | 32.193.164.228 |
| NAT Gateway | xfusion-natgw | nat-04060ff9b89b87619 | In public subnet |
| S3 Bucket | xfusion-nat-3445 | N/A | Target upload bucket |

---

## Step-by-Step Implementation

### Step 1: Discover Existing VPC Resources

Retrieve and export the IDs for the existing VPC and private subnet. These variables are used throughout all subsequent steps.

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=xfusion-priv-vpc" \
  --query "Vpcs[0].VpcId" \
  --output text)

PRIV_SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=xfusion-priv-subnet" \
  --query "Subnets[0].SubnetId" \
  --output text)

PRIV_AZ=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=xfusion-priv-subnet" \
  --query "Subnets[0].AvailabilityZone" \
  --output text)

echo "VPC_ID=$VPC_ID"
echo "PRIV_SUBNET_ID=$PRIV_SUBNET_ID"
echo "PRIV_AZ=$PRIV_AZ"
```

**Expected output:**

```
VPC_ID=vpc-0cf7a9d60553403ab
PRIV_SUBNET_ID=subnet-0f76f16569c103e13
PRIV_AZ=us-east-1a
```

Confirm the VPC CIDR block and all subnet CIDRs to avoid conflicts before creating new subnets:

```bash
aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query "Vpcs[0].CidrBlock" \
  --output text

aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].{CIDR:CidrBlock,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

**Expected output:**

```
10.1.0.0/16
----------------------------------------
|            DescribeSubnets           |
+--------------+-----------------------+
|     CIDR     |         Name          |
+--------------+-----------------------+
|  10.1.1.0/24 |  xfusion-priv-subnet  |
+--------------+-----------------------+
```

> **Screenshot**

<img width="1027" height="807" alt="image" src="https://github.com/user-attachments/assets/cbf561a7-eae6-472d-b22b-6b74e44e5512" />

> `Terminal output showing VPC_ID, PRIV_SUBNET_ID, PRIV_AZ variables and subnet table`

---

### Step 2: Create a Public Subnet

Create a public subnet in the same Availability Zone as the private subnet. Using the same AZ (`us-east-1a`) ensures the NAT Gateway is in the same zone as the private EC2 instance, minimizing cross-AZ data transfer costs.

```bash
PUB_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.1.2.0/24 \
  --availability-zone us-east-1a \
  --query "Subnet.SubnetId" \
  --output text)

echo "PUB_SUBNET_ID=$PUB_SUBNET_ID"
```

Tag the subnet for operational clarity:

```bash
aws ec2 create-tags \
  --resources $PUB_SUBNET_ID \
  --tags Key=Name,Value=xfusion-pub-subnet
```

Verify the new subnet:

```bash
aws ec2 describe-subnets \
  --subnet-ids $PUB_SUBNET_ID \
  --query "Subnets[0].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

**Expected output:**

```
---------------------------------------------------------------------------------
|                                DescribeSubnets                                |
+------------+--------------+----------------------------+----------------------+
|     AZ     |    CIDR      |            ID              |        Name          |
+------------+--------------+----------------------------+----------------------+
|  us-east-1a|  10.1.2.0/24 |  subnet-0779d195127e0e801  |  xfusion-pub-subnet  |
+------------+--------------+----------------------------+----------------------+
```

> **Screenshot**

<img width="1027" height="547" alt="image" src="https://github.com/user-attachments/assets/3bc58222-ec53-4747-8059-07a7dbc7b48a" />

> `Terminal showing PUB_SUBNET_ID and the subnet details table with CIDR 10.1.2.0/24`

---

### Step 3: Create and Attach an Internet Gateway

An Internet Gateway (IGW) is the VPC component that enables communication between the VPC and the internet. It is required for the public subnet and for the NAT Gateway to route outbound traffic.

**Create the Internet Gateway:**

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --query "InternetGateway.InternetGatewayId" \
  --output text)

echo "IGW_ID=$IGW_ID"
```

**Tag it:**

```bash
aws ec2 create-tags \
  --resources $IGW_ID \
  --tags Key=Name,Value=xfusion-igw
```

**Attach to the VPC:**

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
```

**Verify the attachment state is `available`:**

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids $IGW_ID \
  --query "InternetGateways[0].{State:Attachments[0].State,VPC:Attachments[0].VpcId,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

**Expected output:**

```
-------------------------------------------------------
|              DescribeInternetGateways               |
+--------------+------------+-------------------------+
|     Name     |   State    |           VPC           |
+--------------+------------+-------------------------+
|  xfusion-igw |  available |  vpc-0cf7a9d60553403ab  |
+--------------+------------+-------------------------+
```

> **Screenshot**

<img width="1026" height="782" alt="image" src="https://github.com/user-attachments/assets/105ed8ac-441d-4997-a8aa-011dd47b1ad2" />

> `Terminal output showing IGW State as "available" and attached VPC ID`

---

### Step 4: Create a Public Route Table

The public route table directs all non-local traffic (0.0.0.0/0) to the Internet Gateway. This is what makes the public subnet truly public.

**Create the route table:**

```bash
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query "RouteTable.RouteTableId" \
  --output text)

echo "PUB_RT_ID=$PUB_RT_ID"
```

**Tag it:**

```bash
aws ec2 create-tags \
  --resources $PUB_RT_ID \
  --tags Key=Name,Value=xfusion-pub-rt
```

**Add the default route to the Internet Gateway:**

```bash
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
```

**Expected output:**

```json
{
    "Return": true
}
```

**Associate the route table with the public subnet:**

```bash
aws ec2 associate-route-table \
  --route-table-id $PUB_RT_ID \
  --subnet-id $PUB_SUBNET_ID
```

**Expected output:**

```json
{
    "AssociationId": "rtbassoc-05b15e5f0eb416ef1",
    "AssociationState": {
        "State": "associated"
    }
}
```

**Verify routes and associations:**

```bash
aws ec2 describe-route-tables \
  --route-table-ids $PUB_RT_ID \
  --query "RouteTables[0].{Routes:Routes[*].{Dest:DestinationCidrBlock,Target:GatewayId,State:State},Associations:Associations[*].{Subnet:SubnetId}}" \
  --output table
```

**Expected output:**

```
------------------------------------------------------
|                 DescribeRouteTables                |
+----------------------------------------------------+
||                   Associations                   ||
|+--------------------------------------------------+|
||                      Subnet                      ||
|+--------------------------------------------------+|
||  subnet-0779d195127e0e801                        ||
|+----------------------------------------------------+|
||                      Routes                      ||
|+--------------+---------+-------------------------+|
||     Dest     |  State  |         Target          ||
|+--------------+---------+-------------------------+|
||  10.1.0.0/16 |  active |  local                  ||
||  0.0.0.0/0   |  active |  igw-066f3fb13549eba8e  ||
|+--------------+---------+-------------------------+|
```

> **Screenshot**

<img width="1036" height="610" alt="image" src="https://github.com/user-attachments/assets/6dedf271-8981-4bef-b037-b7fffab86369" />
<img width="1032" height="799" alt="image" src="https://github.com/user-attachments/assets/1122cdd8-b7c0-46a6-970d-26dd4616b589" />

> `Route table output showing 0.0.0.0/0 route targeting the IGW and subnet association`

---

### Step 5: Allocate an Elastic IP and Create the NAT Gateway

The NAT Gateway requires a static public IP address (Elastic IP) to maintain a consistent outbound identity. The NAT Gateway is deployed into the **public subnet** so it can access the internet via the IGW.

**Allocate an Elastic IP:**

```bash
EIP_ALLOC_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query "AllocationId" \
  --output text)

echo "EIP_ALLOC_ID=$EIP_ALLOC_ID"
```

**Expected output:**

```
EIP_ALLOC_ID=eipalloc-08e5e560f682cfe6a
```

**Create the NAT Gateway in the public subnet:**

```bash
NATGW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_ID \
  --allocation-id $EIP_ALLOC_ID \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=xfusion-natgw}]' \
  --query "NatGateway.NatGatewayId" \
  --output text)

echo "NATGW_ID=$NATGW_ID"
```

**Wait for the NAT Gateway to become available (this typically takes 60 to 90 seconds):**

```bash
aws ec2 wait nat-gateway-available \
  --nat-gateway-ids $NATGW_ID

echo "NAT Gateway ready"
```

**Verify the NAT Gateway state and Elastic IP:**

```bash
aws ec2 describe-nat-gateways \
  --nat-gateway-ids $NATGW_ID \
  --query "NatGateways[0].{State:State,Name:Tags[?Key=='Name']|[0].Value,Subnet:SubnetId,EIP:NatGatewayAddresses[0].PublicIp}" \
  --output table
```

**Expected output:**

```
------------------------------------------------------------------------------
|                             DescribeNatGateways                            |
+----------------+----------------+------------+-----------------------------+
|       EIP      |     Name       |   State    |           Subnet            |
+----------------+----------------+------------+-----------------------------+
|  32.193.164.228|  xfusion-natgw |  available |  subnet-0779d195127e0e801   |
+----------------+----------------+------------+-----------------------------+
```

> **Screenshot**

<img width="1032" height="750" alt="image" src="https://github.com/user-attachments/assets/33ba0439-33b7-43a5-bbb0-b8f4da550e75" />

> `NAT Gateway details showing State "available", EIP, and public subnet placement`

---

### Step 6: Create a Private Route Table and Route via NAT Gateway

Update the private subnet's routing so that all outbound traffic is forwarded to the NAT Gateway. This is the critical step that enables the private EC2 instance to reach the internet without being directly exposed.

**Create the private route table:**

```bash
PRIV_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --query "RouteTable.RouteTableId" \
  --output text)

echo "PRIV_RT_ID=$PRIV_RT_ID"
```

**Tag it:**

```bash
aws ec2 create-tags \
  --resources $PRIV_RT_ID \
  --tags Key=Name,Value=xfusion-priv-rt
```

**Add the default route targeting the NAT Gateway:**

```bash
aws ec2 create-route \
  --route-table-id $PRIV_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NATGW_ID
```

**Expected output:**

```json
{
    "Return": true
}
```

**Associate the private route table with the private subnet:**

```bash
aws ec2 associate-route-table \
  --route-table-id $PRIV_RT_ID \
  --subnet-id $PRIV_SUBNET_ID
```

**Expected output:**

```json
{
    "AssociationId": "rtbassoc-0bd88716b475f03b1",
    "AssociationState": {
        "State": "associated"
    }
}
```

**Verify the private route table:**

```bash
aws ec2 describe-route-tables \
  --route-table-ids $PRIV_RT_ID \
  --query "RouteTables[0].{Routes:Routes[*].{Dest:DestinationCidrBlock,NatGW:NatGatewayId,State:State},Associations:Associations[*].{Subnet:SubnetId}}" \
  --output table
```

**Expected output:**

```
------------------------------------------------------
|                 DescribeRouteTables                |
+----------------------------------------------------+
||                   Associations                   ||
|+--------------------------------------------------+|
||                      Subnet                      ||
|+--------------------------------------------------+|
||  subnet-0f76f16569c103e13                        ||
|+--------------------------------------------------+|
||                      Routes                      ||
|+--------------+-------------------------+---------+|
||     Dest     |          NatGW          |  State  ||
|+--------------+-------------------------+---------+|
||  10.1.0.0/16 |  None                   |  active ||
||  0.0.0.0/0   |  nat-04060ff9b89b87619  |  active ||
|+--------------+-------------------------+---------+|
```

> **Screenshot**



> `Private route table showing 0.0.0.0/0 route targeting the NAT Gateway ID and private subnet association`

---

### Step 7: Verify End-to-End Connectivity

The private EC2 instance (`xfusion-priv-ec2`) has a cron job configured to upload a file to the S3 bucket `xfusion-nat-3445` once internet access is established. After completing all configuration steps, wait **2 to 3 minutes** for the cron job to execute, then verify:

```bash
aws s3 ls s3://xfusion-nat-3445/
```

**Expected output confirming successful upload:**

```
2026-03-20 01:15:04          0 xfusion-test.txt
```

The presence of `xfusion-test.txt` in the bucket confirms that the private EC2 instance successfully reached the internet via the NAT Gateway and uploaded the file to S3.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of "aws s3 ls s3://xfusion-nat-3445/" showing xfusion-test.txt with timestamp]`

---

## Verification and Validation

Run the following commands to perform a complete health check of the architecture after deployment:

```bash
# Confirm VPC and subnet layout
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].{Name:Tags[?Key=='Name']|[0].Value,CIDR:CidrBlock,AZ:AvailabilityZone}" \
  --output table

# Confirm IGW attachment
aws ec2 describe-internet-gateways \
  --internet-gateway-ids $IGW_ID \
  --query "InternetGateways[0].Attachments[0].State" \
  --output text

# Confirm NAT Gateway status
aws ec2 describe-nat-gateways \
  --nat-gateway-ids $NATGW_ID \
  --query "NatGateways[0].State" \
  --output text

# Confirm S3 upload success
aws s3 ls s3://xfusion-nat-3445/
```

> **Screenshot Placeholder**
> `[SCREENSHOT: All four validation commands run sequentially with clean output confirming "available" states and file presence in S3]`

---

## Best Practices

### Networking

* **Same Availability Zone placement:** Always deploy the NAT Gateway in the same AZ as the private subnet instances it serves. Cross-AZ NAT traffic incurs additional data transfer charges and increases latency.
* **Non-overlapping CIDRs:** Before creating any subnet, enumerate all existing subnets in the VPC to prevent CIDR conflicts. The existing `10.1.1.0/24` private subnet was inspected before allocating `10.1.2.0/24` for the public subnet.
* **Separate route tables:** Never share a route table between public and private subnets. Public subnets must route to the IGW; private subnets must route to the NAT Gateway. Mixing these tables is a common misconfiguration that creates security exposure.
* **Tagging every resource:** Apply consistent `Name` tags to all resources immediately after creation. This reduces confusion during incident response, cost attribution, and cleanup operations.

### Security

* **NAT Gateway is outbound only:** The NAT Gateway does not allow unsolicited inbound connections from the internet. Private instances remain unreachable from the public internet, which is the desired security posture.
* **Security Group hygiene:** Ensure the EC2 instance security group allows outbound HTTPS (port 443) to S3 endpoints. Overly restrictive egress rules can silently block traffic even when the route is correct.
* **Elastic IP lifecycle management:** Track allocated Elastic IPs. Unattached EIPs incur hourly charges. If the NAT Gateway is deleted, release the EIP explicitly to avoid unnecessary costs.

### Operational

* **Use `aws ec2 wait` for NAT Gateway provisioning:** NAT Gateways take 60 to 90 seconds to become available. Always use the `wait nat-gateway-available` command rather than polling manually or proceeding too early.
* **Store resource IDs in shell variables:** Exporting IDs as environment variables (`VPC_ID`, `NATGW_ID`, etc.) throughout the session reduces transcription errors and makes commands composable.
* **Verify at every step:** Run describe commands after each resource creation to confirm expected state before proceeding. Catching a misconfiguration early is far less costly than debugging a broken route table after the full stack is deployed.

### Cost Management

* **NAT Gateway pricing:** AWS charges per hour for NAT Gateway availability plus per-GB data processing. For development environments, consider stopping the NAT Gateway outside business hours or using a NAT instance as a lower-cost alternative.
* **S3 VPC Endpoints:** For production workloads where EC2 instances primarily communicate with S3, consider adding an S3 VPC Gateway Endpoint. Traffic to S3 via the endpoint does not pass through the NAT Gateway and incurs no NAT data processing charges.

---

## Lessons Learned

### 1. CIDR Planning Before Subnet Creation

Running a describe on all existing subnets before allocating a new CIDR block prevented a potential overlap conflict. Always audit the existing address space before expanding it. An overlap would cause the subnet creation to fail with a non-obvious error.

### 2. Sequence Dependency is Strict

The order of operations is non-negotiable in this architecture:

```
IGW created --> IGW attached to VPC --> Public Route Table with IGW route
--> Public Subnet associated with Public RT --> NAT Gateway created in Public Subnet
--> Private Route Table with NAT route --> Private Subnet associated with Private RT
```

Attempting to create a NAT Gateway in a subnet without a working IGW route will result in an available NAT Gateway that cannot actually forward traffic.

### 3. The `wait` Command Prevents Race Conditions

Without `aws ec2 wait nat-gateway-available`, any route table operation referencing the NAT Gateway ID while it is still in the `pending` state will fail. This failure mode is subtle because the NAT Gateway ID exists and is valid, but the resource is not yet operational.

### 4. Route Table Association Does Not Replace Default Routes

A new route table created in a VPC automatically receives a local route for the VPC CIDR. This local route cannot be deleted or overridden. Only the default route (0.0.0.0/0) needs to be added manually. Understanding this avoids confusion when verifying routes.

### 5. S3 Upload Verification Is the True End-to-End Test

Verifying that the NAT Gateway state is `available` and that routes exist is necessary but not sufficient. The actual end-to-end test is confirming that the EC2 instance inside the private subnet can reach the internet and perform a real operation, which in this case is the S3 file upload via the cron job. Always drive verification from the application layer, not just the infrastructure layer.

### 6. Tag Resources Immediately

Tagging after the fact risks losing track of resource IDs, especially in multi-resource deployments. Applying tags at creation time (using `--tag-specifications` where the API supports it, as with the NAT Gateway) is preferable to a separate `create-tags` call.

---

## Resource Reference

| Resource | AWS Documentation |
|---|---|
| NAT Gateway | https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html |
| Internet Gateway | https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html |
| Route Tables | https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html |
| Elastic IP Addresses | https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html |
| S3 VPC Endpoints | https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html |
| VPC Subnet CIDR Blocks | https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html |

---








<img width="1032" height="807" alt="image" src="https://github.com/user-attachments/assets/54d569eb-434c-4423-878d-98504d1e39d0" />
<img width="1031" height="492" alt="image" src="https://github.com/user-attachments/assets/ca8a1af6-7cd2-4ea2-9e58-4f564aee949f" />

<img width="1035" height="786" alt="image" src="https://github.com/user-attachments/assets/97c07788-4b41-465b-9903-4a29e1559097" />
<img width="1035" height="618" alt="image" src="https://github.com/user-attachments/assets/c1420252-1f56-42db-89ab-d2d43bbdd960" />
<img width="1032" height="806" alt="image" src="https://github.com/user-attachments/assets/1f625092-cccc-42e3-90d2-7e4b20d79b9e" />
<img width="1034" height="712" alt="image" src="https://github.com/user-attachments/assets/f0a58856-0e27-4f2e-98e3-778843dda05c" />
