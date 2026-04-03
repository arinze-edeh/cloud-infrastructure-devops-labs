# AWS EC2 Security Group Configuration via CLI

**Implementing Network-Level Access Control for Application Infrastructure on AWS**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Step 1: Identify the Target VPC](#step-1-identify-the-target-vpc)
- [Step 2: Create the Security Group](#step-2-create-the-security-group)
- [Step 3: Configure Inbound Firewall Rules](#step-3-configure-inbound-firewall-rules)
- [Step 4: Verify Security Group Configuration](#step-4-verify-security-group-configuration)
- [Result Summary](#result-summary)
- [Security Best Practices](#security-best-practices)
- [Production Considerations](#production-considerations)
- [Troubleshooting](#troubleshooting)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document details the process of provisioning and configuring an AWS EC2 Security Group using the AWS CLI. Security groups act as stateful virtual firewalls that control inbound and outbound traffic at the instance level. Proper configuration is foundational to any secure, production-grade cloud infrastructure.

This workflow reflects the standard approach used by DevOps and Cloud Engineers when standing up new application environments, and is fully reproducible via the AWS CLI for auditability and automation compatibility.

---

## Problem Statement

When deploying application servers on AWS EC2, instances require explicit network access rules before they can serve traffic or accept administrative connections. Without a properly scoped security group, EC2 instances either remain unreachable or are exposed with overly permissive default configurations.

**Goal:** Create a named security group scoped to the application VPC, define the minimum required inbound rules (HTTP and SSH), and validate the configuration is correctly applied before associating it with any EC2 instance.

---

## Architecture Context

| Parameter | Value |
|---|---|
| Cloud Provider | AWS |
| Region | us-east-1 |
| Network | Default VPC |
| Resource Type | EC2 Security Group |
| Access Method | AWS CLI |
| Security Group Name | nautilus-sg |
| Inbound Rules | TCP 80 (HTTP), TCP 22 (SSH) |

---

## Prerequisites

- AWS CLI installed and configured (`aws configure`)
- IAM permissions: `ec2:DescribeVpcs`, `ec2:CreateSecurityGroup`, `ec2:AuthorizeSecurityGroupIngress`, `ec2:DescribeSecurityGroups`
- Active AWS account with access to the target region

---

## Step 1: Identify the Target VPC

Before creating a security group, the target VPC must be identified. Security groups are scoped to a specific VPC and cannot be shared across VPCs.

**Command:**

```bash
aws ec2 describe-vpcs \
  --query "Vpcs[].VpcId" \
  --output table
```

**What this does:** Queries all VPCs in the current region and returns their IDs in a formatted table. In environments with a single default VPC, this returns one entry.

**Expected Output:**

```
--------------------------
|      DescribeVpcs      |
+------------------------+
|  vpc-01ebedea54fdb8b6b |
+------------------------+
```

**Screenshot: VPC ID retrieval and Security Group creation**

<img width="995" height="686" alt="image" src="https://github.com/user-attachments/assets/05781afe-e697-4e8f-ad2b-632c37d1ab87" />


*The VPC ID `vpc-01ebedea54fdb8b6b` is retrieved and immediately used to scope the new security group to the correct network boundary.*

> **Operational Note:** In multi-VPC environments, refine the query with `--filters Name=isDefault,Values=true` to target only the default VPC, or specify the VPC by known ID when working with custom network topologies.

---

## Step 2: Create the Security Group

With the VPC ID confirmed, the security group is created with a descriptive name and associated directly with the target VPC. The group starts with no inbound rules by default.

**Command:**

```bash
aws ec2 create-security-group \
  --group-name nautilus-sg \
  --description "Security group for Nautilus App Servers" \
  --vpc-id vpc-01ebedea54fdb8b6b
```

**What this does:** Creates a new security group inside the specified VPC. AWS automatically applies a default egress rule allowing all outbound traffic. No inbound traffic is permitted until rules are explicitly added.

**Expected Output:**

```json
{
    "GroupId": "sg-0ea00d5ddf9da2bac",
    "SecurityGroupArn": "arn:aws:ec2:us-east-1:825526788326:security-group/sg-0ea00d5ddf9da2bac"
}
```

**Screenshot: Security Group successfully created**

<img width="992" height="679" alt="image" src="https://github.com/user-attachments/assets/42f4783a-1bae-4456-9f6b-b0f4e7175551" />


*The returned `GroupId` (`sg-0ea00d5ddf9da2bac`) is the unique identifier used in all subsequent rule configuration and instance association commands.*

> **Important:** Note the `GroupId` from this output. It is required for all subsequent steps. Using the group name in place of the ID is supported in some commands but the ID is preferred for precision and scripting reliability.

---

## Step 3: Configure Inbound Firewall Rules

Security group rules are applied incrementally. Each `authorize-security-group-ingress` call adds one rule to the group. Two inbound rules are required: one for HTTP web traffic and one for SSH administrative access.

### Rule 1: Allow HTTP Traffic (Port 80)

**Command:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0ea00d5ddf9da2bac \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**Purpose:** Permits inbound TCP traffic on port 80 from any source IP. This enables the server to respond to HTTP requests from the public internet, which is the expected behavior for a publicly accessible web application.

**Screenshot: HTTP and SSH ingress rules authorized**

<img width="995" height="821" alt="image" src="https://github.com/user-attachments/assets/7f17dc21-9229-4a38-8410-54852d751c5b" />


*Both port 80 and port 22 ingress rules are authorized successfully. Each rule returns a `SecurityGroupRuleId` confirming the rule was applied.*

**Expected Output:**

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-035f9db5f4bcb19fa",
            "GroupId": "sg-0ea00d5ddf9da2bac",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

---

### Rule 2: Allow SSH Access (Port 22)

**Command:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0ea00d5ddf9da2bac \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

**Purpose:** Permits inbound TCP traffic on port 22 from any source IP. This enables SSH connections to the server for administrative operations, deployments, and troubleshooting.

**Expected Output:**

```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-049dca42ae5c9bd67",
            "GroupId": "sg-0ea00d5ddf9da2bac",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

> **Security Note:** Opening SSH to `0.0.0.0/0` is acceptable in sandbox or development environments. In production, this CIDR must be restricted to a known static IP or a corporate VPN CIDR. See [Security Best Practices](#security-best-practices) below.

---

## Step 4: Verify Security Group Configuration

Once all rules are applied, the configuration is validated by querying the security group directly. This step confirms that all rules are present, correctly scoped, and associated with the intended VPC before the group is attached to any EC2 instance.

**Command:**

```bash
aws ec2 describe-security-groups \
  --group-ids sg-0ea00d5ddf9da2bac
```

**What this does:** Returns the full specification of the security group including its VPC association, all inbound (`IpPermissions`) rules, and all outbound (`IpPermissionsEgress`) rules.

**Screenshot: Full security group verification output (Part 1)**

![Security Group Describe Output Part 1](https://github.com/user-attachments/assets/02d93e47-5a6e-4c3a-a24b-279a0f6f09eb)

*The describe output confirms the group is linked to `vpc-01ebedea54fdb8b6b`, has the correct name and description, and shows both inbound rules for TCP 80 and TCP 22.*

**Screenshot: Full security group verification output (Part 2)**

![Security Group Describe Output Part 2](https://github.com/user-attachments/assets/17853506-e6a6-44a3-bd10-a3113cdf4b4c)

*Continued output confirming the TCP 22 rule with `CidrIp: "0.0.0.0/0"` is active alongside the HTTP rule.*

**Screenshot: Full security group verification output (Part 3)**

![Security Group Describe Output Part 3](https://github.com/user-attachments/assets/8aeca849-6020-4ae5-b0a3-14dbb7435dec)

*Final section of the describe output confirming the egress rule (all traffic allowed outbound by default) and the complete rule set is intact.*

**Validation Checklist:**

- `GroupName` equals `nautilus-sg`
- `VpcId` matches `vpc-01ebedea54fdb8b6b`
- `IpPermissions` contains two entries: TCP 80 and TCP 22, both with `CidrIp: 0.0.0.0/0`
- `IpPermissionsEgress` contains the default all-traffic outbound rule
- `IsEgress: false` on both inbound rules confirms correct direction

---

## Result Summary

| Component | Status | Detail |
|---|---|---|
| VPC Identified | Complete | `vpc-01ebedea54fdb8b6b` |
| Security Group Created | Complete | `sg-0ea00d5ddf9da2bac` (nautilus-sg) |
| HTTP Rule (TCP 80) | Applied | Source: `0.0.0.0/0` |
| SSH Rule (TCP 22) | Applied | Source: `0.0.0.0/0` |
| Configuration Validated | Passed | All rules confirmed via describe |

The security group `nautilus-sg` is fully configured and ready to be associated with EC2 instances in the Nautilus application environment.

---

## Security Best Practices

**Applied in this configuration:**

- Network access is controlled at the VPC level using a dedicated security group
- Inbound rules are explicit and intentional; no traffic is permitted by default
- The security group is named and described for operational clarity
- CLI-driven configuration ensures repeatability and auditability

**Recommended hardening for production environments:**

- **Restrict SSH access:** Replace `0.0.0.0/0` on port 22 with your corporate VPN CIDR or a specific trusted IP range. Example: `--cidr 10.0.0.0/8` for an internal network
- **Use HTTPS over HTTP:** Add a rule for TCP 443 and terminate TLS at the load balancer or instance. Consider removing port 80 or redirecting to 443
- **Prefer AWS Systems Manager Session Manager:** Eliminates the need for SSH entirely by providing browser-based or CLI shell access without opening port 22
- **Apply the principle of least privilege:** Only open ports that are actively used by the application
- **Tag security groups:** Add tags such as `Environment`, `Owner`, and `Application` for cost allocation and governance

```bash
aws ec2 create-tags \
  --resources sg-0ea00d5ddf9da2bac \
  --tags Key=Environment,Value=production Key=Application,Value=nautilus
```

---

## Production Considerations

**Infrastructure as Code (IaC):** For production deployments, security group configurations should be codified in Terraform, AWS CloudFormation, or AWS CDK rather than applied manually via CLI. This ensures version control, peer review, and consistent state management.

**Automation integration:** The CLI commands in this document are directly scriptable and can be embedded in CI/CD pipelines, bootstrap scripts, or Ansible playbooks for automated environment provisioning.

**Drift detection:** Periodically audit security group rules using AWS Config rules such as `restricted-ssh` and `vpc-default-security-group-closed` to detect unauthorized rule additions.

**Rule revocation:** To remove a rule that is no longer needed:

```bash
aws ec2 revoke-security-group-ingress \
  --group-id sg-0ea00d5ddf9da2bac \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
```

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `create-security-group` returns `InvalidVpcID.NotFound` | Incorrect or miscopied VPC ID | Re-run `describe-vpcs` and confirm the VPC ID |
| `InvalidGroup.Duplicate` on group creation | A group with that name already exists in the VPC | Use `describe-security-groups --filters Name=group-name,Values=nautilus-sg` to locate the existing group |
| `AuthFailure` or `UnauthorizedOperation` | IAM user lacks required permissions | Attach the `AmazonEC2FullAccess` policy or a custom policy with the required `ec2:*` actions |
| EC2 instance not reachable on port 80 | Security group not attached to instance, or instance not running a web server | Verify the security group is associated with the instance via `describe-instances` |
| SSH connection refused | Port 22 rule not applied, or instance's OS-level firewall is active | Confirm the rule exists with `describe-security-groups` and check the instance's `iptables` or `firewalld` status |

---

## Skills Demonstrated

- **AWS Networking Fundamentals:** VPC structure, security group scope, stateful firewall behavior
- **Security Groups and Firewall Rule Management:** Inbound rule creation, CIDR-based access control, rule validation
- **AWS CLI Proficiency:** Multi-step CLI workflows, JMESPath query filtering, structured output parsing
- **Cloud Security Engineering:** Least-privilege access design, production hardening recommendations
- **Infrastructure Validation:** Post-configuration verification, rule auditing via `describe-security-groups`
- **Production Documentation:** Structured runbook format suitable for onboarding, handoff, and incident response































# AWS Security Group Configuration (EC2 Access Control)

## Overview

This lab demonstrates **practical AWS networking security**, focusing on **firewall rules**, **least-privilege access**, and **CLI-based cloud operations**.

---

## Architecture Context

ENVIRONMENT:
- Cloud Provider = AWS
- Region = us-east-1
- Network = Default VPC
- Resource Type = Security Group
- Access Method = AWS CLI


---

## Step 1: Identify the Target VPC


ACTION:
- Query AWS to identify the default VPC
COMMAND:
- aws ec2 describe-vpcs
FILTER isDefault = true
OUTPUT:
- VPC_ID retrieved for security group association


📸 **Screenshot**  
<img width="1048" height="906" alt="image" src="https://github.com/user-attachments/assets/11847b2f-bfdf-48b4-855a-d74363241635" />

---

## Step 2: Create Security Group


ACTION:
 Create a security group inside the default VPC
INPUT PARAMETERS:
- Group Name = nautilus-sg
- Description = "Security group for Nautilus App Servers"
- VPC_ID = <default-vpc-id>
COMMAND:
- aws ec2 create-security-group


EXPECTED RESULT:


OUTPUT:
SecurityGroupId generated successfully


📸 **Screenshot**  
<img width="1048" height="906" alt="image" src="https://github.com/user-attachments/assets/11847b2f-bfdf-48b4-855a-d74363241635" />

---

## Step 3: Configure Inbound Rules (Firewall Rules)


RULE 1:
Protocol = HTTP
Port = 80
Source = 0.0.0.0/0
Purpose = Allow public web traffic

RULE 2:
Protocol = SSH
Port = 22
Source = 0.0.0.0/0
Purpose = Enable administrative access


COMMANDS:


aws ec2 authorize-security-group-ingress


📸 **Screenshot**  
<img width="1030" height="909" alt="image" src="https://github.com/user-attachments/assets/5618d954-e849-4621-9fbd-63eef140042d" />

---

## Step 4: Verify Security Group Configuration


ACTION:
Retrieve and validate security group rules
COMMAND:
aws ec2 describe-security-groups
VALIDATION:
Confirm:
- HTTP (80) rule exists
- SSH (22) rule exists
- Correct CIDR ranges applied


📸 **Screenshot**  
<img width="1075" height="901" alt="image" src="https://github.com/user-attachments/assets/02d93e47-5a6e-4c3a-a24b-279a0f6f09eb" />
<img width="1044" height="880" alt="image" src="https://github.com/user-attachments/assets/17853506-e6a6-44a3-bd10-a3113cdf4b4c" />
<img width="1060" height="892" alt="image" src="https://github.com/user-attachments/assets/8aeca849-6020-4ae5-b0a3-14dbb7435dec" />

---

## Result


RESULT:
- EC2 traffic is securely controlled via Security Group
- Only required ports are exposed
- Security group is ready for EC2 instance attachment


---

## Security Best Practices Applied


- Network access controlled at the VPC level
- Explicit inbound rules defined
- Security group scoped to application use case
- CLI used for repeatable, auditable configuration


> NOTE: In production environments, SSH access should be restricted to trusted IP ranges instead of `0.0.0.0/0`.

---

## Real-World Relevance


USE CASES:
- Securing web application servers
- Enforcing network-level access controls
- Preparing infrastructure for scalable EC2 deployments


This mirrors **real DevOps and Cloud Engineer workflows** used in production AWS environments.

---

## Skills Demonstrated


- AWS Networking Fundamentals
- Security Groups & Firewall Rules
- AWS CLI Proficiency
- Cloud Security Best Practices
- Infrastructure Validation & Troubleshooting


---

## Recruiter Summary


This lab proves hands-on experience with:
- AWS network security
- CLI-driven infrastructure management
- Production-aligned cloud access control


---
