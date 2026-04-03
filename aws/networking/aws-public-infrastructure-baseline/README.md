# AWS Public Infrastructure Baseline: VPC, Internet Gateway, and EC2 Provisioning

> **Status:** Completed | **Region:** us-east-1 | **Baseline Ready for Extension:** Yes

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Scope and Objectives](#scope-and-objectives)
- [Prerequisites and Environment](#prerequisites-and-environment)
- [Implementation](#implementation)
  - [Step 1: Create Public VPC](#step-1-create-public-vpc)
  - [Step 2: Create Internet Gateway](#step-2-create-internet-gateway)
  - [Step 3: Attach Internet Gateway to VPC](#step-3-attach-internet-gateway-to-vpc)
  - [Step 4: Create Public Subnet](#step-4-create-public-subnet)
  - [Step 5: Enable Auto-Assign Public IP on Subnet](#step-5-enable-auto-assign-public-ip-on-subnet)
  - [Step 6: Create Route Table](#step-6-create-route-table)
  - [Step 7: Add Internet Route](#step-7-add-internet-route)
  - [Step 8: Associate Route Table with Public Subnet](#step-8-associate-route-table-with-public-subnet)
  - [Step 9: Create Security Group](#step-9-create-security-group)
  - [Step 10: Allow Inbound SSH](#step-10-allow-inbound-ssh)
  - [Step 11: Launch EC2 Instance](#step-11-launch-ec2-instance)
  - [Step 12: Validate Public Accessibility](#step-12-validate-public-accessibility)
- [Validation Checklist](#validation-checklist)
- [Key Learnings](#key-learnings)
- [Operational Considerations and Best Practices](#operational-considerations-and-best-practices)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Cleanup Procedure](#cleanup-procedure)

---

## Project Overview

This document provides a **production-grade reference implementation** for provisioning a foundational public AWS network topology using the AWS CLI. The infrastructure establishes a custom Virtual Private Cloud (VPC) with internet-facing compute capability, forming a reusable baseline for cloud-native and hybrid workloads deployed to AWS.

All resources are provisioned programmatically via the AWS CLI in the **us-east-1** region, following the principle of infrastructure-as-code and explicit resource ownership.

---

## Problem Statement

AWS accounts initialized with default networking configurations carry implicit risks: shared IP spaces, default security group rules, and lack of explicit routing controls. Relying on default VPCs introduces governance gaps and inhibits consistent environment replication across teams.

**This baseline solves the following engineering problems:**

- Eliminates dependence on the default VPC for internet-facing workloads
- Establishes explicit, auditable network topology from first principles
- Provides a controlled, reproducible foundation for multi-environment deployments
- Reduces the blast radius of misconfigurations by isolating compute in purpose-built subnets
- Enforces infrastructure sequencing: network before compute

---

## Architecture Overview

### Logical Components

| Component | Configuration |
|---|---|
| VPC | CIDR: `10.0.0.0/16`, DNS support and hostnames enabled |
| Public Subnet | CIDR: `10.0.1.0/24`, AZ: `us-east-1e`, auto-assign public IP enabled |
| Internet Gateway | Attached to VPC, enables bidirectional internet traffic |
| Route Table | Default route `0.0.0.0/0` targeting the Internet Gateway |
| Security Group | Allows inbound TCP port 22 (SSH) from `0.0.0.0/0` |
| EC2 Instance | `t2.micro`, Amazon Linux AMI, public subnet, public IP assigned |

### Traffic Flow

```
Internet
    |
Internet Gateway (devops-pub-igw)
    |
Route Table (devops-pub-rt) --> 0.0.0.0/0
    |
Public Subnet (10.0.1.0/24, us-east-1e)
    |
EC2 Instance (devops-pub-ec2)
    |
Security Group (devops-ssh-sg) --> TCP 22 allowed
```

---

## Scope and Objectives

- Establish a non-default public VPC with DNS support enabled
- Provision and attach an Internet Gateway for bidirectional internet access
- Create an explicit route table with a default internet route
- Configure a public subnet with automatic public IP assignment
- Deploy a minimal EC2 compute instance for connectivity validation
- Deliver a reusable **network foundation** for future workload extension

---

## Prerequisites and Environment

### Region

```
AWS_REGION = us-east-1
```

### Identity Verification

Before provisioning any resources, verify the active AWS identity to ensure operations are executed under the correct account and IAM principal:

```bash
aws sts get-caller-identity
```

**Expected output fields:** `UserId`, `Account`, `Arn`

> **Operational Note:** Always confirm caller identity at the start of any provisioning session. Unintended resource creation in the wrong account is a common and costly error.

**Screenshot: AWS STS Identity Verification**

![AWS STS Identity Verification](https://github.com/user-attachments/assets/a1ee7e16-c8dc-4fb3-8149-23cdff0c7210)

*Verifying the active IAM identity before beginning resource provisioning ensures operations are scoped to the intended AWS account.*

---

## Implementation

### Step 1: Create Public VPC

**Intent:** Establish an isolated, non-default Virtual Private Cloud with a `/16` CIDR block, providing ample IP space for subnet segmentation. DNS support and hostname resolution are explicitly enabled to support future service discovery and private DNS requirements.

```bash
# Create VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=devops-pub-vpc}]'

# Enable DNS support
aws ec2 modify-vpc-attribute \
  --vpc-id <vpc-id> \
  --enable-dns-support

# Enable DNS hostnames
aws ec2 modify-vpc-attribute \
  --vpc-id <vpc-id> \
  --enable-dns-hostnames
```

**Configuration:**

- `cidr_block` = `10.0.0.0/16`
- `name` = `devops-pub-vpc`
- `enable_dns_support` = `true`
- `enable_dns_hostnames` = `true`

> **Best Practice:** Always enable DNS support and hostname resolution on custom VPCs. Many AWS services (RDS, ECS, EFS) depend on internal DNS resolution. Disabling these attributes can cause intermittent service failures that are difficult to diagnose.

**Screenshot: VPC Created Successfully**

![VPC Created Successfully](https://github.com/user-attachments/assets/43e6bc23-6db9-4a9d-aa67-6e957d5db9bc)

*The custom VPC `devops-pub-vpc` has been created with a `10.0.0.0/16` CIDR block. Note the VPC ID returned, which is required for subsequent resource association steps.*

---

### Step 2: Create Internet Gateway

**Intent:** Provision an Internet Gateway (IGW) as the entry and exit point for all internet-bound traffic originating from or destined to resources within the VPC. An IGW is a horizontally scaled, redundant, and highly available AWS-managed component.

```bash
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=devops-pub-igw}]'
```

**Configuration:**

- `name` = `devops-pub-igw`

> **Note:** An Internet Gateway is stateless and does not perform NAT. It enables bidirectional communication between instances in a public subnet and the internet, provided that instances have a public IP and routing is correctly configured.

**Screenshot: Internet Gateway Created**

![Internet Gateway Created](https://github.com/user-attachments/assets/a1364c50-6851-45f4-9211-3d01a5850a3c)

*The Internet Gateway `devops-pub-igw` has been created and is in a detached state. Attachment to the VPC is performed in the next step.*

---

### Step 3: Attach Internet Gateway to VPC

**Intent:** Bind the Internet Gateway to the target VPC, establishing the connectivity path between the VPC's public subnets and the internet. A VPC can have only one IGW attached at a time.

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>
```

**Configuration:**

- Attaching `devops-pub-igw` to `devops-pub-vpc`

> **Edge Case:** If an IGW attachment attempt fails, verify that the VPC does not already have an IGW attached. Each VPC supports exactly one Internet Gateway. Attempting to attach a second will return an error.

**Screenshot: Internet Gateway Attached to VPC**

![Internet Gateway Attached to VPC](https://github.com/user-attachments/assets/22631e7f-2c25-49b0-9d6e-e79a6a4c855a)

*The Internet Gateway is now attached to `devops-pub-vpc`, transitioning its state from `detached` to `available`. This enables internet routing once route tables are configured.*

---

### Step 4: Create Public Subnet

**Intent:** Provision a subnet within the VPC scoped to a single Availability Zone. This subnet will house public-facing compute resources. The `/24` CIDR provides 251 usable IP addresses (5 are reserved by AWS).

```bash
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1e \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=devops-pub-subnet}]'
```

**Configuration:**

- `vpc` = `devops-pub-vpc`
- `cidr_block` = `10.0.1.0/24`
- `availability_zone` = `us-east-1e`
- `name` = `devops-pub-subnet`

> **Operational Consideration:** For production workloads, distribute subnets across multiple Availability Zones to achieve high availability. A single-AZ subnet is appropriate for baseline validation but represents a single point of failure for live traffic.

**Screenshot: Subnet Created**

![Subnet Created](https://github.com/user-attachments/assets/129745cc-9ef4-4bc6-b506-a0b90a49f9ec)

*The public subnet `devops-pub-subnet` with CIDR `10.0.1.0/24` has been created in `us-east-1e` within the custom VPC. Note the Subnet ID for subsequent association steps.*

---

### Step 5: Enable Auto-Assign Public IP on Subnet

**Intent:** Configure the subnet to automatically assign a public IPv4 address to any EC2 instance launched within it. This eliminates the need to manually specify public IP assignment at instance launch time and ensures consistent internet reachability for all instances in this subnet.

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id <subnet-id> \
  --map-public-ip-on-launch
```

**Configuration:**

- `subnet` = `devops-pub-subnet`
- `map_public_ip_on_launch` = `true`

> **Best Practice:** Enable public IP auto-assignment only on subnets explicitly designated as public. Never enable this attribute on private subnets, as it exposes instances to the internet without explicit intent and can bypass security controls.

**Screenshot: Public IP Auto-Assignment Enabled**

![Public IP Auto-Assignment Enabled](https://github.com/user-attachments/assets/021e1a60-62a2-4129-959d-5166a72a6e29)

*The subnet attribute `MapPublicIpOnLaunch` is now set to `true`. All instances launched in this subnet will receive a public IPv4 address automatically.*

---

### Step 6: Create Route Table

**Intent:** Create a dedicated route table for the public subnet. Using a custom route table rather than the VPC's main route table provides explicit routing control and prevents unintended internet exposure of other subnets that implicitly inherit the main route table.

```bash
aws ec2 create-route-table \
  --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=devops-pub-rt}]'
```

**Configuration:**

- `vpc` = `devops-pub-vpc`
- `name` = `devops-pub-rt`

> **Best Practice:** Never add internet routes to the VPC's **main route table**. Doing so inadvertently exposes all subnets that have not been explicitly associated with a custom route table, including private and management subnets.

**Screenshot: Route Table Created**

![Route Table Created](https://github.com/user-attachments/assets/97adc556-204d-4c3c-a6b8-55275af9e0a2)

*A dedicated route table `devops-pub-rt` has been created within `devops-pub-vpc`. The default local route (`10.0.0.0/16`) is automatically added and cannot be removed.*

---

### Step 7: Add Internet Route

**Intent:** Add a default route (`0.0.0.0/0`) to the route table that directs all non-local traffic to the Internet Gateway. This is the critical routing rule that transforms an ordinary subnet into a public subnet.

```bash
aws ec2 create-route \
  --route-table-id <route-table-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <igw-id>
```

**Configuration:**

- `route_table` = `devops-pub-rt`
- `destination` = `0.0.0.0/0`
- `target` = Internet Gateway (`devops-pub-igw`)

> **Technical Clarification:** A subnet is only considered "public" when its associated route table contains a route to an Internet Gateway with destination `0.0.0.0/0`. Without this explicit route entry, even instances with public IPs cannot reach the internet.

**Screenshot: Default Route Added**

![Default Route Added](https://github.com/user-attachments/assets/c5b00172-c77e-4e07-a372-be8995c44522)

*The default route `0.0.0.0/0 -> igw-xxxxxxxx` has been successfully added to `devops-pub-rt`. All traffic not destined for the local VPC CIDR will now be forwarded to the Internet Gateway.*

---

### Step 8: Associate Route Table with Public Subnet

**Intent:** Explicitly bind the custom route table to the public subnet. Without this association, the subnet falls back to using the VPC's main route table, which lacks the internet route and would prevent outbound connectivity.

```bash
aws ec2 associate-route-table \
  --route-table-id <route-table-id> \
  --subnet-id <subnet-id>
```

**Configuration:**

- `route_table` = `devops-pub-rt`
- `subnet` = `devops-pub-subnet`

> **Troubleshooting Insight:** If an instance in the subnet cannot reach the internet despite having a public IP, always verify the subnet-to-route-table association first. A missing or incorrect association is one of the most common networking misconfiguration issues in AWS.

**Screenshot: Route Table Association**

![Route Table Association](https://github.com/user-attachments/assets/2b204198-b322-43d7-891b-ac1d4928e5c6)

*The route table `devops-pub-rt` is now explicitly associated with `devops-pub-subnet`. The subnet is now fully public, with internet routing enforced through the Internet Gateway.*

---

### Step 9: Create Security Group

**Intent:** Provision a security group scoped to the VPC to act as a virtual firewall for EC2 instances. Security groups are stateful, meaning return traffic for allowed inbound connections is automatically permitted without an explicit outbound rule.

```bash
aws ec2 create-security-group \
  --group-name devops-ssh-sg \
  --description "Allow SSH access" \
  --vpc-id <vpc-id>
```

**Configuration:**

- `name` = `devops-ssh-sg`
- `vpc` = `devops-pub-vpc`
- `description` = `allow ssh access`

> **Best Practice:** Always provide a meaningful description for security groups. In environments with many groups, descriptions are the primary mechanism for identifying the purpose of a security group without inspecting its rules.

**Screenshot: Security Group Created**

![Security Group Created](https://github.com/user-attachments/assets/c9868ac8-2178-47e5-ab6f-f3330eb537bd)

*Security group `devops-ssh-sg` has been created within the VPC. By default, a new security group allows all outbound traffic and denies all inbound traffic until rules are explicitly added.*

---

### Step 10: Allow Inbound SSH

**Intent:** Add an ingress rule to permit inbound TCP traffic on port 22 (SSH) from any source IP. This allows direct SSH access to EC2 instances assigned this security group for administrative and validation purposes.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**Configuration:**

- `security_group` = `devops-ssh-sg`
- `protocol` = `tcp`
- `port` = `22`
- `source` = `0.0.0.0/0`

> **Security Risk:** Allowing SSH from `0.0.0.0/0` (the entire internet) is acceptable for short-lived validation environments but is **not recommended for production**. In production, restrict the source CIDR to known IP ranges, use AWS Systems Manager Session Manager as a keyless alternative, or place a bastion host in front of private compute.

**Screenshot: SSH Rule Configured**

![SSH Rule Configured](https://github.com/user-attachments/assets/4e5cabc6-3ffa-4683-b341-7879a0df17a9)

*The inbound rule permitting TCP port 22 from `0.0.0.0/0` has been successfully applied to `devops-ssh-sg`. The security group is now configured to accept SSH connections from any source.*

---

### Step 11: Launch EC2 Instance

**Intent:** Deploy a `t2.micro` EC2 instance using the Amazon Linux AMI into the public subnet with the SSH security group attached. This instance serves as the compute validation target to confirm end-to-end network configuration, internet connectivity, and public IP assignment.

```bash
aws ec2 run-instances \
  --image-id <amazon-linux-ami-id> \
  --instance-type t2.micro \
  --subnet-id <subnet-id> \
  --security-group-ids <sg-id> \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=devops-pub-ec2}]'
```

**Configuration:**

- `name` = `devops-pub-ec2`
- `ami` = Amazon Linux AMI
- `instance_type` = `t2.micro`
- `subnet` = `devops-pub-subnet`
- `security_group` = `devops-ssh-sg`
- `assign_public_ip` = `true`

> **Operational Note:** Network infrastructure (VPC, subnets, IGW, route tables, security groups) must be fully provisioned and validated before launching compute. Launching an instance into an incomplete network topology results in instances that are unreachable and may require termination and reprovisioning.

**Screenshot: EC2 Instance Launch Initiated (1 of 4)**

![EC2 Instance Launch Initiated 1](https://github.com/user-attachments/assets/ddafa25b-4f06-4bd5-9050-2c9d76f80f98)

*The `run-instances` CLI command has been submitted. The instance enters a `pending` state as AWS allocates underlying hardware and network resources.*

**Screenshot: EC2 Instance Launch Initiated (2 of 4)**

![EC2 Instance Launch Initiated 2](https://github.com/user-attachments/assets/686f19e0-2e38-4430-be8c-d42f00dea3ae)

*The instance provisioning output confirms the AMI, instance type, subnet, and security group assignments. Verify these values against the intended configuration before proceeding.*

**Screenshot: EC2 Instance Launch Initiated (3 of 4)**

![EC2 Instance Launch Initiated 3](https://github.com/user-attachments/assets/022da898-e348-4b82-ac52-88a9f6a028ff)

*Additional instance metadata is returned, including the private IP address assigned within the `10.0.1.0/24` subnet range. The public IP will be assigned once the instance reaches a `running` state.*

**Screenshot: EC2 Instance Launch Initiated (4 of 4)**

![EC2 Instance Launch Initiated 4](https://github.com/user-attachments/assets/f8c6a09f-c37c-49a0-861f-42c2b979936c)

*The instance launch response confirms successful placement in the target subnet and Availability Zone. Allow 30 to 60 seconds for the instance to reach `running` state before attempting connectivity validation.*

---

### Step 12: Validate Public Accessibility

**Intent:** Retrieve the public IPv4 address assigned to the EC2 instance and confirm that all prior provisioning steps have resulted in a fully internet-accessible compute resource. This step serves as the final end-to-end validation of the entire infrastructure stack.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=devops-pub-ec2" \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,State.Name]' \
  --output table
```

**Configuration:**

- `filter` = `name:devops-pub-ec2`
- `retrieve` = `public_ip`

**Verification Steps:**

1. Confirm the instance state is `running`
2. Confirm a public IP address has been assigned
3. Optionally, attempt an SSH connection to validate port 22 reachability:

```bash
ssh -i <key.pem> ec2-user@<public-ip>
```

**Screenshot: Public IP Assigned**

![Public IP Assigned](https://github.com/user-attachments/assets/9bad9d59-f2d2-4505-bc54-675b3850df24)

*The EC2 instance `devops-pub-ec2` is in a `running` state with a public IPv4 address assigned. This confirms successful end-to-end provisioning: VPC, IGW, routing, subnet, and compute are all correctly configured and operational.*

---

## Validation Checklist

- **VPC** is non-default, active, and has DNS support and hostname resolution enabled
- **Internet Gateway** is created, attached to the VPC, and in an `available` state
- **Route Table** contains the local VPC route and a `0.0.0.0/0` route targeting the Internet Gateway
- **Route Table** is explicitly associated with the public subnet
- **Subnet** auto-assigns public IPs on instance launch
- **Security Group** has an inbound rule permitting TCP port 22
- **EC2 Instance** is in a `running` state with a public IPv4 address assigned
- **SSH connectivity** to the instance is reachable from the expected source

---

## Key Learnings

- **Public subnets require explicit routing:** A subnet is not public by virtue of having a public IP assigned. It requires an Internet Gateway attachment and a `0.0.0.0/0` route in its associated route table.
- **Public IP assignment is a two-layer control:** It can be set at the subnet level (auto-assign) or overridden at instance launch. Both layers must be aligned with intent.
- **Infrastructure sequencing is non-negotiable:** Network components (VPC, IGW, subnets, route tables, security groups) must be fully provisioned before compute. Launching instances into an incomplete network results in unreachable resources.
- **Security groups are stateful firewalls:** Return traffic for allowed sessions is automatically permitted. Only inbound rules need to be explicitly defined for SSH and similar administrative protocols.
- **Custom route tables prevent blast radius expansion:** Keeping internet routes out of the main route table protects subnets that have not been explicitly designated as public.
- **A baseline VPC simplifies scaling and governance:** A well-structured foundational VPC dramatically reduces time-to-deploy for future workloads and enforces consistent networking standards across teams.

---

## Operational Considerations and Best Practices

- **Restrict SSH access by IP:** Replace the `0.0.0.0/0` SSH source CIDR with the specific IP range of your administrative network or VPN egress. This significantly reduces the attack surface.
- **Use key pairs:** Always associate an EC2 key pair at launch time for secure, auditable SSH authentication. Store private keys in a secrets manager, not on local filesystems.
- **Tag all resources consistently:** Apply standardized tags (Environment, Owner, Project, CostCenter) to every resource from the start. Retroactive tagging is error-prone and often incomplete.
- **Enable VPC Flow Logs:** Activate Flow Logs on the VPC immediately after creation. They provide critical visibility into accepted and rejected traffic and are essential for security incident response.
- **Prefer Systems Manager Session Manager:** For production environments, eliminate the need for SSH port 22 entirely by using AWS Systems Manager Session Manager, which provides audited shell access without inbound port exposure.
- **Use Elastic IPs for persistent addressing:** Public IPs assigned at launch are ephemeral and change on stop/start cycles. Allocate and associate an Elastic IP if a stable, persistent public address is required.
- **Multi-AZ distribution:** Replicate this subnet and compute pattern across at least two Availability Zones for any workload with uptime requirements.

---

## Risks and Edge Cases

| Risk | Impact | Mitigation |
|---|---|---|
| SSH open to `0.0.0.0/0` | High: brute-force and unauthorized access exposure | Restrict to known IP ranges or use SSM Session Manager |
| Single Availability Zone deployment | Medium: AZ-level failure causes full outage | Distribute subnets and instances across multiple AZs |
| Ephemeral public IP on EC2 | Low: IP changes after stop/start | Allocate and associate an Elastic IP |
| No VPC Flow Logs enabled | Medium: no traffic audit trail | Enable Flow Logs to S3 or CloudWatch immediately |
| Missing resource tags | Low: cost attribution and governance gaps | Enforce tagging via AWS Config rules or tag policies |
| Key pair not associated at launch | High: instance is permanently inaccessible via SSH | Always specify a key pair or configure SSM before launch |

---

## Cleanup Procedure

Execute the following steps in order to avoid dependency errors during resource deletion. AWS enforces resource association constraints, and out-of-order deletion will result in errors.

```bash
# 1. Terminate EC2 instance
aws ec2 terminate-instances --instance-ids <instance-id>

# Wait for instance to reach 'terminated' state before proceeding
aws ec2 wait instance-terminated --instance-ids <instance-id>

# 2. Delete security group
aws ec2 delete-security-group --group-id <sg-id>

# 3. Disassociate and delete route table
aws ec2 disassociate-route-table --association-id <association-id>
aws ec2 delete-route-table --route-table-id <route-table-id>

# 4. Detach and delete Internet Gateway
aws ec2 detach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>

# 5. Delete subnet
aws ec2 delete-subnet --subnet-id <subnet-id>

# 6. Delete VPC
aws ec2 delete-vpc --vpc-id <vpc-id>
```

> **Important:** The VPC cannot be deleted until all dependent resources (instances, subnets, route tables, security groups, and the IGW) have been fully removed. Always verify each deletion step completes successfully before proceeding to the next.

---

*Document authored to production-grade engineering standards. Suitable for enterprise onboarding, operational handoff, and infrastructure knowledge base publication.*
























# AWS Public Infrastructure Baseline (VPC + Internet-Facing EC2)

## Project Overview
This project documents the creation of a **public AWS infrastructure baseline** designed to support internet-facing workloads. The baseline includes a custom VPC, a public subnet with automatic public IP assignment, an Internet Gateway, routing configuration, a security group allowing SSH access, and a publicly accessible EC2 instance.  

All resources are provisioned using the AWS CLI in the **us-east-1** region.

---

## Scope and Objectives
- Establish a non-default public VPC
- Enable outbound and inbound internet connectivity
- Enforce explicit routing via Internet Gateway
- Provision a minimal EC2 compute resource for validation
- Serve as a reusable **network foundation** for future workloads

---

## Architecture Overview

### Logical Components
- VPC (10.0.0.0/16)
- Public Subnet (10.0.1.0/24)
- Internet Gateway
- Route Table with 0.0.0.0/0 route
- Security Group (SSH access)
- EC2 Instance (t2.micro)
  
---

## Environment and Preconditions

### Region
```
AWS_REGION = us-east-1
Identity Verification
aws sts get-caller-identity
```
Screenshot: `AWS STS Identity Verification`
<img width="1034" height="530" alt="image" src="https://github.com/user-attachments/assets/a1ee7e16-c8dc-4fb3-8149-23cdff0c7210" />

## Implementation Steps

## Step 1: Create Public VPC
```
create vpc
cidr_block = 10.0.0.0/16
name = devops-pub-vpc
enable_dns_support = true
enable_dns_hostnames = true
```
Screenshot: `VPC Created Successfully`
<img width="1028" height="839" alt="image" src="https://github.com/user-attachments/assets/43e6bc23-6db9-4a9d-aa67-6e957d5db9bc" />

## Step 2: Create Internet Gateway

```
create internet_gateway
name = devops-pub-igw
```
Screenshot: `Internet Gateway Created`
<img width="1032" height="652" alt="image" src="https://github.com/user-attachments/assets/a1364c50-6851-45f4-9211-3d01a5850a3c" />

## Step 3: Attach Internet Gateway to VPC
```
attach internet_gateway
to vpc = devops-pub-vpc
```
Screenshot: `Internet Gateway Attached to VPC`

<img width="1034" height="451" alt="image" src="https://github.com/user-attachments/assets/22631e7f-2c25-49b0-9d6e-e79a6a4c855a" />

## Step 4: Create Public Subnet
```
create subnet
vpc = devops-pub-vpc
cidr_block = 10.0.1.0/24
availability_zone = us-east-1e
name = devops-pub-subnet
```
Screenshot: `Subnet Created`
<img width="1031" height="857" alt="image" src="https://github.com/user-attachments/assets/129745cc-9ef4-4bc6-b506-a0b90a49f9ec" />

## Step 5: Enable Auto-Assign Public IP on Subnet
```
modify subnet attribute
subnet = devops-pub-subnet
map_public_ip_on_launch = true
```
Screenshot: `Public IP Auto-Assignment Enabled`
<img width="1036" height="515" alt="image" src="https://github.com/user-attachments/assets/021e1a60-62a2-4129-959d-5166a72a6e29" />

## Step 6: Create Route Table
```
create route_table
vpc = devops-pub-vpc
name = devops-pub-rt
```
Screenshot: `Route Table Created`
<img width="1032" height="705" alt="image" src="https://github.com/user-attachments/assets/97adc556-204d-4c3c-a6b8-55275af9e0a2" />

## Step 7: Add Internet Route
```
add route
route_table = devops-pub-rt
destination = 0.0.0.0/0
target = internet_gateway
```
Screenshot: `Default Route Added`
<img width="1029" height="787" alt="image" src="https://github.com/user-attachments/assets/c5b00172-c77e-4e07-a372-be8995c44522" />

## Step 8: Associate Route Table with Public Subnet
```
associate route_table
route_table = devops-pub-rt
subnet = devops-pub-subnet
```
Screenshot: `Route Table Association`
<img width="1035" height="418" alt="image" src="https://github.com/user-attachments/assets/2b204198-b322-43d7-891b-ac1d4928e5c6" />

## Step 9: Create Security Group (SSH Access)
```
create security_group
name = devops-ssh-sg
vpc = devops-pub-vpc
description = allow ssh access
```
Screenshot: `Security Group Created`
<img width="1030" height="537" alt="image" src="https://github.com/user-attachments/assets/c9868ac8-2178-47e5-ab6f-f3330eb537bd" />

## Step 10: Allow Inbound SSH
```
authorize ingress
security_group = devops-ssh-sg
protocol = tcp
port = 22
source = 0.0.0.0/0
```
Screenshot: `SSH Rule Configured`
<img width="1036" height="595" alt="image" src="https://github.com/user-attachments/assets/4e5cabc6-3ffa-4683-b341-7879a0df17a9" /> 

## Step 11: Launch EC2 Instance
```
run ec2_instance
name = devops-pub-ec2
ami = amazon_linux_ami
instance_type = t2.micro
subnet = devops-pub-subnet
security_group = devops-ssh-sg
assign_public_ip = true
```
Screenshots: `EC2 Instance Launch Initiated`
<img width="1029" height="854" alt="image" src="https://github.com/user-attachments/assets/ddafa25b-4f06-4bd5-9050-2c9d76f80f98" />
<img width="1020" height="863" alt="image" src="https://github.com/user-attachments/assets/686f19e0-2e38-4430-be8c-d42f00dea3ae" />
<img width="1030" height="862" alt="image" src="https://github.com/user-attachments/assets/022da898-e348-4b82-ac52-88a9f6a028ff" />
<img width="1035" height="482" alt="image" src="https://github.com/user-attachments/assets/f8c6a09f-c37c-49a0-861f-42c2b979936c" />

## Step 12: Validate Public Accessibility
```
describe ec2_instance
filter = name:devops-pub-ec2
retrieve public_ip
```
Screenshot: `Public IP Assigned`
<img width="1040" height="801" alt="image" src="https://github.com/user-attachments/assets/9bad9d59-f2d2-4505-bc54-675b3850df24" />

## Validation Checklist

- VPC is non-default and active

- Subnet auto-assigns public IPs

- Route table contains 0.0.0.0/0 → IGW

- EC2 instance has a public IP

- SSH port 22 reachable from the internet

## Key Learnings

- Public subnets require explicit routing to an Internet Gateway

- Public IP assignment must be enabled at subnet or instance level

- Network infrastructure should be provisioned before compute

- A baseline VPC simplifies future scaling and governance

## Cleanup Procedure
- terminate ec2_instance
- delete security_group
- disassociate and delete route_table
- detach and delete internet_gateway
- delete subnet
- delete vpc

## Status
- PROJECT_STATUS = `completed`
- BASELINE_READY_FOR_EXTENSION = `true`















