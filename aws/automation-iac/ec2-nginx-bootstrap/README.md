# Automated EC2 Nginx Web Server Provisioning via AWS CLI

**Category:** Cloud Infrastructure | AWS | DevOps Automation
**Region:** us-east-1
**Skill Level:** Intermediate
**Toolchain:** AWS CLI, Ubuntu, Nginx, Bash

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Step 1: Validate AWS Identity](#step-1-validate-aws-identity)
- [Step 2: Identify Default VPC](#step-2-identify-default-vpc)
- [Step 3: Create Security Group](#step-3-create-security-group)
- [Step 4: Allow HTTP Ingress](#step-4-allow-http-ingress)
- [Step 5: Create User Data Script](#step-5-create-user-data-script)
- [Step 6: Launch EC2 Instance](#step-6-launch-ec2-instance)
- [Step 7: Retrieve Public IP Address](#step-7-retrieve-public-ip-address)
- [Step 8: Validate Nginx Deployment](#step-8-validate-nginx-deployment)
- [Validation Checklist](#validation-checklist)
- [Operational Notes](#operational-notes)
- [Troubleshooting and Edge Cases](#troubleshooting-and-edge-cases)
- [Cleanup](#cleanup)

---

## Overview

This document details a **fully CLI-driven provisioning workflow** for deploying an Ubuntu EC2 instance configured as a publicly accessible Nginx web server on AWS. Every step is performed using the **AWS CLI**, eliminating console dependency and enabling reproducible, scriptable infrastructure deployments consistent with modern DevOps and Infrastructure-as-Code (IaC) principles.

This workflow is designed to serve as a reference implementation for teams automating EC2-based web server deployments in controlled, auditable environments.

---

## Problem Statement

Manual infrastructure provisioning through the AWS Management Console introduces inconsistency, is difficult to audit, and cannot be reliably reproduced across environments or team members. Clickops workflows are unsuitable for production pipelines.

**Solution:** Implement a fully automated, CLI-driven provisioning process that:

- Validates identity and credentials before any resource creation
- Establishes a properly scoped security group with minimal ingress exposure
- Bootstraps an Nginx web server via EC2 user data at instance launch
- Verifies deployment success through HTTP response validation

---

## Architecture

```
Internet
   |
Port 80 (HTTP)
   |
AWS Security Group (xfusion-web-sg)
   |
EC2 Instance -- Ubuntu 22.04 LTS (t2.micro)
   |
Nginx Web Server (systemd-managed)
```

**Key Design Decisions:**

- Security group permits only port 80 inbound, following the principle of least privilege
- Nginx is installed and enabled at boot via user data, ensuring idempotent bootstrapping
- No SSH key pair is attached, reducing the attack surface for this Nginx-only workload
- Instance is tagged for deterministic resource querying via CLI filters

---

## Prerequisites

Ensure the following are in place before beginning:

- **AWS CLI v2** installed and configured (`aws --version`)
- **Valid IAM credentials** with permissions for EC2 and VPC operations
- **Default region** set to `us-east-1` in your AWS config or via environment variable
- **Bash shell** available for user data script creation and curl validation

```bash
# Verify AWS CLI is configured
aws configure list

# Confirm active region
aws configure get region
```

---

## Step 1: Validate AWS Identity

**Intent:** Confirm that the CLI is operating under the correct IAM principal before provisioning any resources. This prevents accidental resource creation in the wrong account, a common and costly mistake in multi-account environments.

```bash
aws sts get-caller-identity
```

**Expected Output Fields:**

- `UserId` -- The IAM user or role identifier
- `Account` -- The 12-digit AWS account ID
- `Arn` -- The full ARN of the calling principal

**Best Practice:** Cross-reference the returned `Account` ID against your target account before proceeding. In CI/CD pipelines, fail the build if the account ID does not match an expected value.

**Screenshot -- Confirmed AWS identity and active account:**

<img width="1029" height="531" alt="AWS STS get-caller-identity output confirming correct IAM user and account" src="https://github.com/user-attachments/assets/aef74ccc-011d-4f36-9670-7090f92c1f20" />

---

## Step 2: Identify Default VPC

**Intent:** Retrieve the default VPC in the target region. The security group created in the next step must be associated with a VPC, and using the default VPC keeps this workflow self-contained without requiring custom network configuration.

```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[*].{VpcId:VpcId,CidrBlock:CidrBlock,IsDefault:IsDefault}" \
  --output table
```

**Note the VPC ID** returned in the output. You will reference this in Step 3. If no default VPC exists, one must be created or an existing VPC ID must be substituted.

**Edge Case:** If your account's default VPC has been deleted (common in hardened AWS environments), create a new one with:

```bash
aws ec2 create-default-vpc
```

**Screenshot -- Default VPC identified in us-east-1:**

<img width="1034" height="809" alt="aws ec2 describe-vpcs output showing the default VPC and its CIDR block" src="https://github.com/user-attachments/assets/b3c344a2-e8d0-4172-9192-d9143cdc0ffd" />

---

## Step 3: Create Security Group

**Intent:** Create a dedicated security group scoped to this workload. Using a workload-specific security group enforces separation of concerns and allows for independent lifecycle management, including teardown without affecting other resources.

```bash
aws ec2 create-security-group \
  --group-name xfusion-web-sg \
  --description "Security group for xfusion-ec2 Nginx server" \
  --vpc-id <VPC_ID>
```

Replace `<VPC_ID>` with the value retrieved in Step 2.

**Store the returned `GroupId`** -- it is required for the ingress rule in Step 4 and the instance launch in Step 6.

**Operational Note:** Avoid using the default security group for application workloads. Workload-specific groups allow precise ingress/egress control and simplify security audits.

**Screenshot -- Security group created with returned GroupId:**

<img width="1030" height="863" alt="aws ec2 create-security-group output showing the new GroupId for xfusion-web-sg" src="https://github.com/user-attachments/assets/4fa86e2f-b54a-48f3-a2aa-3283a96b0484" />

---

## Step 4: Allow HTTP Ingress

**Intent:** Open port 80 to all inbound traffic (`0.0.0.0/0`) so the Nginx server is publicly accessible over HTTP. This is the minimal ingress rule required for a public-facing web server.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id <SECURITY_GROUP_ID> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

**Security Consideration:** In production, restrict the source CIDR to known IP ranges or place the instance behind a load balancer with a WAF. Exposing port 80 globally is acceptable for this demonstration environment but should be re-evaluated for sensitive workloads.

**Risk:** No HTTPS (port 443) is configured. For production use, terminate TLS at a load balancer or configure Nginx with a certificate via Let's Encrypt.

**Screenshot -- Ingress rule authorizing TCP port 80 from any source:**

<img width="1030" height="640" alt="aws ec2 authorize-security-group-ingress output confirming port 80 rule applied" src="https://github.com/user-attachments/assets/d38ece55-85ec-40d4-96e7-954c7576b47f" />

---

## Step 5: Create User Data Script

**Intent:** Define an automated bootstrap script that runs on first boot of the EC2 instance. This ensures Nginx is installed, started, and enabled for automatic restart on reboot, without requiring any manual SSH access post-launch.

```bash
cat > userdata.sh << 'EOF'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
EOF
```

**Why This Approach Works:**

- `apt-get update -y` ensures the package index is refreshed before installation
- `apt-get install -y nginx` installs Nginx non-interactively
- `systemctl start nginx` brings the service up immediately after installation
- `systemctl enable nginx` registers Nginx to start automatically on subsequent reboots

**Best Practice:** Validate the script locally before embedding it in a production launch. For complex bootstrapping, consider AWS Systems Manager (SSM) or cloud-init configurations for better observability and error handling.

**Screenshot -- userdata.sh created with Nginx bootstrap commands:**

<img width="1030" height="620" alt="Terminal showing contents of userdata.sh with nginx install and systemctl commands" src="https://github.com/user-attachments/assets/5c89fc1b-48ed-4015-a4e6-3492cbd52859" />

---

## Step 6: Launch EC2 Instance

**Intent:** Provision the EC2 instance with the correct AMI, instance type, security group, user data script, and identifying tag. The tag `Name=xfusion-ec2` is critical for deterministic resource querying in all subsequent steps.

```bash
aws ec2 run-instances \
  --image-id <UBUNTU_AMI_ID> \
  --instance-type t2.micro \
  --security-group-ids <SECURITY_GROUP_ID> \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=xfusion-ec2}]' \
  --count 1
```

**Key Parameters:**

- `--image-id` -- Use the latest Ubuntu 22.04 LTS AMI for us-east-1. Retrieve the current AMI ID via the SSM Parameter Store to avoid hardcoding:

```bash
aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/22.04/stable/current/amd64/hvm/ebs-gp2/ami-id \
  --query "Parameter.Value" \
  --output text
```

- `--instance-type t2.micro` -- Eligible for AWS Free Tier. Scale to `t3.small` or larger for production workloads
- `--user-data file://userdata.sh` -- Passes the bootstrap script directly at launch
- Tag naming is mandatory for subsequent CLI filtering in Step 7

**Note:** The instance will enter a `pending` state before transitioning to `running`. User data execution begins during the boot sequence and may take 30 to 60 seconds to complete after the instance reaches `running` status.

**Screenshots -- EC2 run-instances command output and instance metadata:**

<img width="1034" height="864" alt="aws ec2 run-instances command output showing instance launch parameters" src="https://github.com/user-attachments/assets/5f62c78b-1fa6-4672-acd6-a3d477d49bba" />
<img width="1030" height="865" alt="Instance metadata output including InstanceId, state, and security groups" src="https://github.com/user-attachments/assets/081e38d3-5a83-4c03-b3bd-15b072496700" />
<img width="1025" height="859" alt="Continued instance metadata showing AMI, launch time, and tags" src="https://github.com/user-attachments/assets/3a1fc5dd-a25a-41a8-b0a7-8d1ea35df23d" />
<img width="1028" height="859" alt="Final instance metadata block confirming network configuration and instance type" src="https://github.com/user-attachments/assets/e31f38fa-c820-452f-bf13-1adc28021f08" />

---

## Step 7: Retrieve Public IP Address

**Intent:** Extract the public IPv4 address of the running instance using the Name tag applied at launch. This approach decouples retrieval from the InstanceId, enabling tag-based lookups that are portable across automation scripts.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
            "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].PublicIpAddress" \
  --output text
```

**Operational Note:** Wait for the instance to reach `running` state before querying the public IP. Use the following command to poll until ready:

```bash
aws ec2 wait instance-running \
  --filters "Name=tag:Name,Values=xfusion-ec2"
```

**Screenshot -- Public IP address extracted via tag-based describe-instances filter:**

<img width="1029" height="547" alt="aws ec2 describe-instances output with PublicIpAddress extracted for xfusion-ec2" src="https://github.com/user-attachments/assets/a52ae395-0248-405d-b52f-967234e880a2" />

---

## Step 8: Validate Nginx Deployment

**Intent:** Confirm end-to-end connectivity and verify that the Nginx web server is actively serving HTTP traffic on port 80. A successful response validates that the security group, user data execution, and Nginx service are all functioning correctly.

```bash
curl -I http://<PUBLIC_IP>
```

**Expected Response:**

```
HTTP/1.1 200 OK
Server: nginx/...
Date: ...
Content-Type: text/html
```

**Validation Logic:**

- `HTTP/1.1 200 OK` -- The server is reachable and Nginx is responding
- `Server: nginx` -- Confirms Nginx is the active HTTP server, not another process
- If the response is `Connection refused`, the instance may still be bootstrapping -- wait 60 seconds and retry
- If the request times out, verify the security group ingress rule and the instance's public IP assignment

**Screenshot -- curl -I confirming HTTP 200 OK and nginx Server header:**

<img width="1032" height="357" alt="curl -I output showing HTTP/1.1 200 OK and Server: nginx response headers" src="https://github.com/user-attachments/assets/81672a03-be32-4869-a386-595f7ec949cb" />

---

## Validation Checklist

| Validation Check | Status |
| --- | --- |
| EC2 instance name is `xfusion-ec2` | PASS |
| Instance running in `us-east-1` | PASS |
| Security group allows HTTP port 80 | PASS |
| User data executed successfully | PASS |
| Nginx service active and enabled | PASS |
| Web server reachable from internet | PASS |

---

## Operational Notes

**Reproducibility**

Every step in this workflow is performed via AWS CLI, making the full deployment scriptable and executable in CI/CD pipelines without modification. The user data script ensures consistent server state at every launch.

**Idempotency**

The user data bootstrap script is safe to re-execute on a fresh instance. `apt-get install -y` is non-destructive if Nginx is already present, and `systemctl enable` is idempotent.

**Minimal Attack Surface**

No SSH key pair is attached to this instance by design. Administrative access, if required, should be configured via AWS Systems Manager Session Manager, which eliminates the need for open port 22 and inbound SSH rules.

**Tagging Strategy**

All resources are tagged at creation. In production environments, extend tags to include `Environment`, `Owner`, `CostCenter`, and `Project` for full operational governance.

---

## Troubleshooting and Edge Cases

**Nginx not responding after curl:**
- Confirm the instance is in `running` state
- Wait up to 90 seconds for user data to complete execution
- Check user data execution logs via Session Manager: `cat /var/log/cloud-init-output.log`

**Security group rule not applying:**
- Verify the correct `GroupId` was used in `authorize-security-group-ingress`
- Confirm the instance is associated with `xfusion-web-sg` via `describe-instances`

**No public IP assigned:**
- Ensure the subnet in the default VPC has `Auto-assign public IPv4` enabled
- Alternatively, allocate and associate an Elastic IP after launch

**AMI not found or deprecated:**
- Use the SSM parameter approach documented in Step 6 to always retrieve a current, non-deprecated AMI ID

---

## Cleanup

Remove all provisioned resources when the deployment is no longer needed to avoid incurring unnecessary charges.

**Terminate EC2 Instance:**

```bash
aws ec2 terminate-instances \
  --instance-ids $(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=xfusion-ec2" \
    --query "Reservations[*].Instances[*].InstanceId" \
    --output text)
```

**Wait for termination before deleting the security group:**

```bash
aws ec2 wait instance-terminated \
  --filters "Name=tag:Name,Values=xfusion-ec2"
```

**Delete Security Group:**

```bash
aws ec2 delete-security-group --group-id <SECURITY_GROUP_ID>
```

**Remove Key Pair (if applicable):**

```bash
aws ec2 delete-key-pair --key-name <KEY_NAME>
```

**Operational Note:** Security groups cannot be deleted while attached to a running or stopped instance. Always terminate the instance and confirm its `terminated` state before attempting security group deletion.

---
















# EC2 Nginx Web Server Deployment via AWS CLI

## Project Overview

This project documents a **CLI-driven provisioning of an Ubuntu EC2 instance** configured as a public-facing **Nginx web server**. The setup uses AWS CLI commands, security groups, and EC2 user data to achieve a fully automated and reproducible infrastructure workflow in the **us-east-1** region. This aligns with DevOps best practices for infrastructure automation and validation.

---

## Architecture Summary

```
Internet
   |
Port 80 (HTTP)
   |
AWS Security Group (xfusion-web-sg)
   |
EC2 Instance (Ubuntu)
   |
Nginx Web Server
```

---

## Prerequisites

```
AWS CLI installed
Valid AWS credentials configured
Region set to us-east-1
```

---

## Step 1: Validate AWS Identity

```
EXECUTE aws sts get-caller-identity
CONFIRM correct Account ID and IAM user
```

### Screenshot:

<img width="1029" height="531" alt="image" src="https://github.com/user-attachments/assets/aef74ccc-011d-4f36-9670-7090f92c1f20" />

---

## Step 2: Identify Default VPC

```
EXECUTE aws ec2 describe-vpcs
CONFIRM default VPC is available
NOTE VPC ID for security group creation
```

### Screenshot:

<img width="1034" height="809" alt="image" src="https://github.com/user-attachments/assets/b3c344a2-e8d0-4172-9192-d9143cdc0ffd" />

---

## Step 3: Create Security Group

```
EXECUTE aws ec2 create-security-group
SET group-name = xfusion-web-sg
SET description = Security group for xfusion-ec2 Nginx server
ASSOCIATE with default VPC
STORE Security Group ID
```

### Screenshot:

<img width="1030" height="863" alt="image" src="https://github.com/user-attachments/assets/4fa86e2f-b54a-48f3-a2aa-3283a96b0484" />

---

## Step 4: Allow HTTP Ingress

```
EXECUTE aws ec2 authorize-security-group-ingress
ALLOW TCP port 80
SOURCE = 0.0.0.0/0
```

### Screenshot:

<img width="1030" height="640" alt="image" src="https://github.com/user-attachments/assets/d38ece55-85ec-40d4-96e7-954c7576b47f" />

---

## Step 5: Create User Data Script

```
CREATE file userdata.sh
ADD the following:

#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
```

### Screenshot:

<img width="1030" height="620" alt="image" src="https://github.com/user-attachments/assets/5c89fc1b-48ed-4015-a4e6-3492cbd52859" />

---

## Step 6: Launch EC2 Instance

```
EXECUTE aws ec2 run-instances
SET AMI = Ubuntu
SET instance-type = t2.micro
ATTACH security group xfusion-web-sg
ATTACH user-data file userdata.sh
TAG instance Name=xfusion-ec2
```

### Screenshots:

<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/5f62c78b-1fa6-4672-acd6-a3d477d49bba" />
<img width="1030" height="865" alt="image" src="https://github.com/user-attachments/assets/081e38d3-5a83-4c03-b3bd-15b072496700" />
<img width="1025" height="859" alt="image" src="https://github.com/user-attachments/assets/3a1fc5dd-a25a-41a8-b0a7-8d1ea35df23d" />
<img width="1028" height="859" alt="image" src="https://github.com/user-attachments/assets/e31f38fa-c820-452f-bf13-1adc28021f08" />

---

## Step 7: Retrieve Public IP Address

```
EXECUTE aws ec2 describe-instances
FILTER by tag Name=xfusion-ec2
EXTRACT PublicIpAddress
```

### Screenshot:

<img width="1029" height="547" alt="image" src="https://github.com/user-attachments/assets/a52ae395-0248-405d-b52f-967234e880a2" />

---

## Step 8: Validate Nginx Deployment

```
EXECUTE curl -I http://<public-ip>
CONFIRM HTTP/1.1 200 OK
CONFIRM Server: nginx
```

### Screenshot:

<img width="1032" height="357" alt="image" src="https://github.com/user-attachments/assets/81672a03-be32-4869-a386-595f7ec949cb" />

---

## Validation Checklist


| Validation Check                   | Status |
| ---------------------------------- | ------ |
| EC2 instance name is `xfusion-ec2` | ✅      |
| Instance running in `us-east-1`    | ✅      |
| Security group allows HTTP (80)    | ✅      |
| User data executed successfully    | ✅      |
| Nginx service active               | ✅      |
| Web server reachable from internet | ✅      |


---

## Operational Notes

```
Deployment performed entirely via AWS CLI
User data ensures idempotent server bootstrap
Security group enforces minimal public access
```

---

## Cleanup (Optional)

```
TERMINATE EC2 instance
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Name,Values=xfusion-ec2" --query "Reservations[*].Instances[*].InstanceId" --output text)

DELETE security group

aws ec2 delete-security-group --group-id sg-03934e8f38d1fb1ed

REMOVE key pair if unused

aws ec2 delete-key-pair --key-name [KEY_NAME]

```
