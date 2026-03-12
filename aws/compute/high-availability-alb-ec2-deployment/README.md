# AWS ALB + EC2 Nginx Web Server Deployment

> **Project:** Nautilus Development Team Infrastructure Provisioning
> **Environment:** AWS us-east-1
> **Stack:** EC2 (Ubuntu 22.04) + Application Load Balancer + Nginx
> **Outcome:** Internet-facing Nginx web server accessible via ALB DNS

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Resource Inventory](#resource-inventory)
- [Implementation](#implementation)
  - [Phase 1: Environment Validation and Discovery](#phase-1-environment-validation-and-discovery)
  - [Phase 2: Security Group Creation](#phase-2-security-group-creation)
  - [Phase 3: User Data Script](#phase-3-user-data-script)
  - [Phase 4: EC2 Instance Launch](#phase-4-ec2-instance-launch)
  - [Phase 5: Application Load Balancer](#phase-5-application-load-balancer)
  - [Phase 6: Target Group and Registration](#phase-6-target-group-and-registration)
  - [Phase 7: ALB Listener Configuration](#phase-7-alb-listener-configuration)
  - [Phase 8: Security Group Adjustments](#phase-8-security-group-adjustments)
  - [Phase 9: Final Verification](#phase-9-final-verification)
- [Known Issues and Resolutions](#known-issues-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [Quick Reference](#quick-reference)

---

## Overview

This runbook documents the end-to-end provisioning of a load-balanced EC2 web server on AWS. It covers the creation of all supporting infrastructure including security groups, an Application Load Balancer, a target group, and an HTTP listener. Nginx is installed and started automatically via a user data bootstrap script at instance launch.

The deployment follows a strict pre-flight discovery pattern: all resource IDs are collected before any infrastructure is created. This eliminates mid-deployment failures caused by dependency resolution errors.

---

## Architecture

```
Internet
    |
    | (port 80, 0.0.0.0/0)
    v
+---------------------------+
|  Application Load Balancer |   xfusion-alb
|  Security Group: default   |   sg-03539665823fc6953
|  Subnets: us-east-1e       |
|           us-east-1d       |
+---------------------------+
    |
    | (port 80, from default SG only)
    v
+---------------------------+
|  Target Group: xfusion-tg  |   HTTP:80, health check: /
+---------------------------+
    |
    v
+---------------------------+
|  EC2 Instance: xfusion-ec2 |   t2.micro, Ubuntu 22.04
|  Security Group: xfusion-sg|   AZ: us-east-1e
|  Nginx: port 80            |   subnet-0c1ddf8bfdedde947
+---------------------------+
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| AWS CLI | v1.40.19 or later |
| IAM Permissions | EC2 full, ELBv2 (create/describe/register), SG management |
| Region | us-east-1 |
| Account | 683588789756 |
| Default VPC | Must exist in target account |

> **Note:** The IAM user in this deployment does **not** have `elasticloadbalancing:SetSubnets` permission. This has direct implications for AZ placement strategy. See [Known Issues and Resolutions](#known-issues-and-resolutions).

---

## Resource Inventory

| Resource | Name | ID |
|---|---|---|
| VPC | default | `vpc-09a4156f3d5e1c2e8` |
| Security Group (EC2) | xfusion-sg | `sg-012c205af0c41893a` |
| Security Group (ALB) | default | `sg-03539665823fc6953` |
| EC2 Instance | xfusion-ec2 | `i-0ea7cfa8ef2a7756d` |
| Subnet (EC2 + ALB) | us-east-1e | `subnet-0c1ddf8bfdedde947` |
| Subnet (ALB) | us-east-1d | `subnet-0885a17fd0dc2df78` |
| AMI | Ubuntu 22.04 LTS | `ami-04680790a315cd58d` |
| Load Balancer | xfusion-alb | `arn:.../app/xfusion-alb/51a4c3075a861f0a` |
| Target Group | xfusion-tg | `arn:.../targetgroup/xfusion-tg/38a2c3614324beec` |
| Listener | HTTP:80 | `arn:.../e05591297333655e` |
| ALB DNS | | `xfusion-alb-1906114081.us-east-1.elb.amazonaws.com` |

---

## Implementation

### Phase 1: Environment Validation and Discovery

Collect all required resource IDs before creating any infrastructure. Do not proceed past this phase until every value in the reference card is populated.

**Step 1.1: Verify AWS CLI**

```bash
aws --version
```

**Step 1.2: Confirm identity and account**

```bash
aws sts get-caller-identity
```

Expected output must show `"Account": "683588789756"`. Stop immediately if the account does not match.

**Step 1.3: Confirm region**

```bash
aws configure get region
```

Expected: `us-east-1`. If not set:

```bash
aws configure set region us-east-1
```

> ***Screenshot: Phase 1 Steps 1.1 to 1.3 terminal output showing CLI version, caller identity JSON, and region confirmation***
<img width="1032" height="480" alt="image" src="https://github.com/user-attachments/assets/bffb71ad-3777-408d-bbee-4148826ee143" />

**Step 1.4: Get Default VPC ID**

```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --region us-east-1 \
  --query "Vpcs[0].VpcId" \
  --output text
```

Result: `vpc-09a4156f3d5e1c2e8`

> ***Screenshot: VPC ID output***
<img width="1030" height="576" alt="image" src="https://github.com/user-attachments/assets/7c57e56d-f09c-4823-ba3c-0d03cd775ef3" />

**Step 1.5: List all subnets in the default VPC**

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-09a4156f3d5e1c2e8" \
  --region us-east-1 \
  --query "Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]" \
  --output table
```

Six subnets returned across six AZs:

| Subnet ID | AZ | CIDR |
|---|---|---|
| subnet-0c1ddf8bfdedde947 | us-east-1e | 172.31.48.0/20 |
| subnet-0885a17fd0dc2df78 | us-east-1d | 172.31.80.0/20 |
| subnet-0de048f2c05528480 | us-east-1b | 172.31.32.0/20 |
| subnet-0324ab7bcaeae4ad5 | us-east-1c | 172.31.0.0/20 |
| subnet-0f72d1d4b29fd2ebb | us-east-1a | 172.31.16.0/20 |
| subnet-0287ca0a3d2eea673 | us-east-1f | 172.31.64.0/20 |

> ***Screenshot: Subnet table output showing all six subnets with AZ and CIDR***
<img width="1030" height="800" alt="image" src="https://github.com/user-attachments/assets/7f2dad58-6350-4bc0-a3ce-b3316ed1596d" />

**Step 1.6: Get Default Security Group ID**

```bash
aws ec2 describe-security-groups \
  --filters \
    "Name=group-name,Values=default" \
    "Name=vpc-id,Values=vpc-09a4156f3d5e1c2e8" \
  --region us-east-1 \
  --query "SecurityGroups[0].GroupId" \
  --output text
```

Result: `sg-03539665823fc6953`

**Step 1.7: Get Latest Ubuntu 22.04 AMI**

```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
    "Name=state,Values=available" \
    "Name=architecture,Values=x86_64" \
  --region us-east-1 \
  --query "sort_by(Images, &CreationDate)[-1].ImageId" \
  --output text
```

Result: `ami-04680790a315cd58d`

> ***Screenshot: Default SG ID and AMI ID outputs***
<img width="1035" height="732" alt="image" src="https://github.com/user-attachments/assets/f9329412-2d21-425f-aa59-ef7687daab35" />

---

### Phase 2: Security Group Creation

**Step 2.1: Create xfusion-sg**

```bash
aws ec2 create-security-group \
  --group-name xfusion-sg \
  --description "xfusion EC2 security group - allows HTTP port 80 from default SG" \
  --vpc-id vpc-09a4156f3d5e1c2e8 \
  --region us-east-1 \
  --query "GroupId" \
  --output text
```

Result: `sg-012c205af0c41893a`

**Step 2.2: Verify the security group was created**

```bash
aws ec2 describe-security-groups \
  --group-ids sg-012c205af0c41893a \
  --region us-east-1 \
  --query "SecurityGroups[0].[GroupId,GroupName,Description]" \
  --output table
```

**Step 2.3: Add inbound rule, port 80 from default SG only**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-012c205af0c41893a \
  --protocol tcp \
  --port 80 \
  --source-group sg-03539665823fc6953 \
  --region us-east-1
```

Expected response includes `"Return": true` and `"FromPort": 80` with `ReferencedGroupInfo` pointing to `sg-03539665823fc6953`. This restricts inbound access to traffic originating from the ALB only. No direct internet access to EC2 is permitted.

> ***Screenshots: Security group creation and ingress rule output showing Return true and ReferencedGroupInfo***
<img width="1035" height="599" alt="image" src="https://github.com/user-attachments/assets/29da420d-b411-4812-b350-963e0b2d0776" />
<img width="1033" height="794" alt="image" src="https://github.com/user-attachments/assets/2b4808ae-4977-4659-a2a0-4bdd10b3e6eb" />

---

### Phase 3: User Data Script

**Step 3.1: Write the bootstrap script to disk**

```bash
cat > /tmp/xfusion-userdata.sh << 'SCRIPT'
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl start nginx
systemctl enable nginx
SCRIPT
```

**Step 3.2: Verify file contents**

```bash
cat /tmp/xfusion-userdata.sh
```

**Step 3.3: Verify Unix line endings**

```bash
cat -A /tmp/xfusion-userdata.sh
```

Every line must end with `$` only. The presence of `^M$` indicates Windows CRLF line endings which will cause the user data script to fail silently at runtime.

> ***Screenshots: cat -A output confirming clean Unix line endings with $ termination on each line***
<img width="1040" height="293" alt="image" src="https://github.com/user-attachments/assets/42363477-1f1c-473c-91b4-eb1af925db69" />
<img width="1034" height="390" alt="image" src="https://github.com/user-attachments/assets/bd5c5f52-94f9-4a09-bf7f-b356c7d70a5b" />
<img width="1035" height="372" alt="image" src="https://github.com/user-attachments/assets/08f9ec6e-4f69-4f8e-91ba-d421aecd02c4" />

---

### Phase 4: EC2 Instance Launch

> **Critical:** Always specify `--subnet-id` explicitly. AWS auto-placement does not guarantee the instance lands in an AZ covered by the ALB. See [Known Issues and Resolutions](#known-issues-and-resolutions) for the full post-mortem on this failure.

**Step 4.1: Launch the instance into the ALB-covered subnet**

```bash
aws ec2 run-instances \
  --image-id ami-04680790a315cd58d \
  --instance-type t2.micro \
  --security-group-ids sg-012c205af0c41893a \
  --subnet-id subnet-0c1ddf8bfdedde947 \
  --user-data file:///tmp/xfusion-userdata.sh \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=xfusion-ec2}]' \
  --region us-east-1 \
  --query "Instances[0].InstanceId" \
  --output text
```

Result: `i-0ea7cfa8ef2a7756d`

**Step 4.2: Wait for running state**

```bash
aws ec2 wait instance-running \
  --instance-ids i-0ea7cfa8ef2a7756d \
  --region us-east-1
```

No output indicates success. This command blocks until the instance reaches running state (typically 30 to 60 seconds).

**Step 4.3: Confirm instance state, AZ, and security group**

```bash
aws ec2 describe-instances \
  --instance-ids i-0ea7cfa8ef2a7756d \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].[InstanceId,State.Name,Placement.AvailabilityZone,SubnetId]" \
  --output table
```

Expected: State `running`, AZ `us-east-1e`, Subnet `subnet-0c1ddf8bfdedde947`.

> ***Screenshot: EC2 describe-instances table showing running state, us-east-1e AZ, and correct subnet***
<img width="1037" height="820" alt="image" src="https://github.com/user-attachments/assets/30f40bf2-e37e-45a1-b0ea-3a6e2e0d49da" />
<img width="1032" height="637" alt="image" src="https://github.com/user-attachments/assets/14a24eab-96ba-4ab9-9df5-f6fb6822f1aa" />

---

### Phase 5: Application Load Balancer

**Step 5.1: Create the ALB**

```bash
aws elbv2 create-load-balancer \
  --name xfusion-alb \
  --subnets subnet-0c1ddf8bfdedde947 subnet-0885a17fd0dc2df78 \
  --security-groups sg-03539665823fc6953 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --region us-east-1 \
  --query "LoadBalancers[0].[LoadBalancerArn,DNSName,State.Code]" \
  --output table
```

Initial state will be `provisioning`. Record both the ARN and DNS name from this output.

**Step 5.2: Wait for active state**

```bash
aws elbv2 wait load-balancer-available \
  --load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:683588789756:loadbalancer/app/xfusion-alb/51a4c3075a861f0a \
  --region us-east-1
```

**Step 5.3: Confirm ALB is active**

```bash
aws elbv2 describe-load-balancers \
  --load-balancer-arns arn:aws:elasticloadbalancing:us-east-1:683588789756:loadbalancer/app/xfusion-alb/51a4c3075a861f0a \
  --region us-east-1 \
  --query "LoadBalancers[0].[LoadBalancerName,State.Code,DNSName,SecurityGroups]" \
  --output table
```

Expected: `State.Code = active`, security group = `sg-03539665823fc6953`.

> ***Screenshot: ALB creation output showing provisioning state, then describe-load-balancers output showing active state with DNS name***
<img width="1035" height="732" alt="image" src="https://github.com/user-attachments/assets/e9d9a049-7e20-4e8f-b616-fbb14ca4aa3c" />

---

### Phase 6: Target Group and Registration

**Step 6.1: Create xfusion-tg**

```bash
aws elbv2 create-target-group \
  --name xfusion-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-09a4156f3d5e1c2e8 \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-port 80 \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --region us-east-1 \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text
```

Result: `arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec`

**Step 6.2: Verify target group**

```bash
aws elbv2 describe-target-groups \
  --names xfusion-tg \
  --region us-east-1 \
  --query "TargetGroups[0].[TargetGroupName,Protocol,Port,TargetType,VpcId]" \
  --output table
```

**Step 6.3: Register the EC2 instance**

```bash
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --targets Id=i-0ea7cfa8ef2a7756d,Port=80 \
  --region us-east-1
```

**Step 6.4: Check initial target health**

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --region us-east-1 \
  --query "TargetHealthDescriptions[*].[Target.Id,Target.Port,TargetHealth.State,TargetHealth.Description]" \
  --output table
```

> **Expected at this stage:** State `unused` with description `Target group is not configured to receive traffic from the load balancer`. This is correct behavior before the listener is created and does not indicate a problem.

> ***Screenshots: Target group creation output and describe-target-groups table showing HTTP port 80 instance type***
<img width="1037" height="676" alt="image" src="https://github.com/user-attachments/assets/8fd74b91-568f-4f46-9b7e-8f2b00de257c" />
<img width="1036" height="632" alt="image" src="https://github.com/user-attachments/assets/412d3c3b-863c-41e6-8fcb-db08aa93910b" />

---

### Phase 7: ALB Listener Configuration

**Step 7.1: Create HTTP listener on port 80**

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:loadbalancer/app/xfusion-alb/51a4c3075a861f0a \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --region us-east-1 \
  --query "Listeners[0].[ListenerArn,Protocol,Port]" \
  --output table
```

Result: `arn:aws:elasticloadbalancing:us-east-1:683588789756:listener/app/xfusion-alb/51a4c3075a861f0a/e05591297333655e`

This step completes the forwarding chain: Internet > ALB > Listener > Target Group > EC2.

> ***Screenshot: create-listener output table showing listener ARN, HTTP, and port 80***
<img width="1034" height="666" alt="image" src="https://github.com/user-attachments/assets/d0e76a2f-ff51-4d68-95a4-a88a2dab6c84" />
---

### Phase 8: Security Group Adjustments

**Step 8.1: Audit existing inbound rules on the default SG**

```bash
aws ec2 describe-security-groups \
  --group-ids sg-03539665823fc6953 \
  --region us-east-1 \
  --query "SecurityGroups[0].IpPermissions[*].[IpProtocol,FromPort,ToPort,IpRanges[0].CidrIp]" \
  --output table
```

In this deployment, the default SG had only the self-referencing rule (`-1 None None None`) and no port 80 inbound from the internet. This means internet traffic cannot reach the ALB.

**Step 8.2: Add inbound port 80 from internet**

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-03539665823fc6953 \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

**Step 8.3: Verify rule was applied**

```bash
aws ec2 describe-security-groups \
  --group-ids sg-03539665823fc6953 \
  --region us-east-1 \
  --query "SecurityGroups[0].IpPermissions[*].[IpProtocol,FromPort,ToPort,IpRanges[0].CidrIp]" \
  --output table
```

Expected: Row showing `tcp | 80 | 80 | 0.0.0.0/0` alongside the existing self-reference rule.

> ***Screenshots: Before and after inbound rules on default SG, showing the addition of tcp 80 0.0.0.0/0***
<img width="1023" height="689" alt="image" src="https://github.com/user-attachments/assets/eada55f7-82de-4db0-96d8-52405340a62a" />
<img width="1041" height="719" alt="image" src="https://github.com/user-attachments/assets/9a1224a0-a3e8-4b51-82ed-55d1069bda55" />

---

### Phase 9: Final Verification

**Step 9.1: Confirm target health**

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --region us-east-1 \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]" \
  --output table
```

Expected: `healthy | None`. If `initial` is returned, Nginx is still starting. Wait 60 seconds and re-run.

> ***Screenshot: describe-target-health output showing i-0ea7cfa8ef2a7756d as healthy***
<img width="1034" height="486" alt="image" src="https://github.com/user-attachments/assets/79c165d3-b3c7-4bd5-9d9a-a063ee31b091" />

**Step 9.2: Test Nginx via ALB DNS**

```bash
curl -I http://xfusion-alb-1906114081.us-east-1.elb.amazonaws.com
```

Expected response:

```
HTTP/1.1 200 OK
Content-Type: text/html
Server: nginx/1.18.0 (Ubuntu)
```

> ***Screenshot: curl -I output showing HTTP 200 OK with Server nginx/1.18.0 Ubuntu header***
<img width="1034" height="486" alt="image" src="https://github.com/user-attachments/assets/79c165d3-b3c7-4bd5-9d9a-a063ee31b091" />

**Step 9.3: Full resource audit**

```bash
echo "=== EC2 ===" && \
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=xfusion-ec2" \
  --region us-east-1 \
  --query "Reservations[0].Instances[0].[InstanceId,State.Name,SecurityGroups[0].GroupId]" \
  --output table && \
echo "=== ALB ===" && \
aws elbv2 describe-load-balancers \
  --names xfusion-alb \
  --region us-east-1 \
  --query "LoadBalancers[0].[LoadBalancerName,State.Code,DNSName]" \
  --output table && \
echo "=== Target Group ===" && \
aws elbv2 describe-target-groups \
  --names xfusion-tg \
  --region us-east-1 \
  --query "TargetGroups[0].[TargetGroupName,Port,VpcId]" \
  --output table
```
---

## Known Issues and Resolutions

### Issue 1: Target Stuck in `unused` State Before Listener Creation

**Symptom:**

```
i-0341e7dc257e2428c | unused | Target group is not configured to receive traffic from the load balancer
```

**Cause:** The target group had an EC2 instance registered but no ALB listener existed yet. Without a listener, the ALB has no forwarding rule and the target group reports `unused`.

**Resolution:** This is expected behavior. Proceed to Phase 7 to create the listener. The state transitions to `initial` then `healthy` automatically once the listener is in place.

**Status:** Non-issue. By design.

---

### Issue 2: Target Stuck in `unused` State After Listener Creation

**Symptom:**

```
i-0341e7dc257e2428c | unused | Target is in an Availability Zone that is not enabled for the load balancer
```

**Cause:** The EC2 instance launched into `us-east-1a` via AWS auto-placement. The ALB was created with subnets only in `us-east-1e` and `us-east-1d`. AWS ALB cannot route to targets in AZs not represented by its subnet configuration.

> ***Screenshot: describe-target-health output showing unused state with AZ not enabled error message***
<img width="1037" height="490" alt="image" src="https://github.com/user-attachments/assets/0d971bd5-acd4-469f-b15e-a276fdc50b19" />
<img width="1038" height="476" alt="image" src="https://github.com/user-attachments/assets/148c88eb-dd94-43aa-9bc0-e2e292811697" />

**Attempted Resolution (Failed):**

`elasticloadbalancing:SetSubnets` was attempted to add `us-east-1a` to the ALB. This was blocked by IAM policy:

```
An error occurred (AccessDenied) when calling the SetSubnets operation:
User is not authorized to perform: elasticloadbalancing:SetSubnets
```

> ***Screenshot: AccessDenied error output from set-subnets attempt***
<img width="1035" height="250" alt="image" src="https://github.com/user-attachments/assets/55f07338-d5ba-4239-bf87-8daaf139142d" />

**Actual Resolution Applied:**

1. Terminate the misplaced EC2 instance
2. Relaunch with an explicit `--subnet-id` pointing to `subnet-0c1ddf8bfdedde947` (us-east-1e), which is one of the ALB's registered subnets
3. Deregister the old instance from the target group
4. Register the new instance

```bash
# Terminate
aws ec2 terminate-instances \
  --instance-ids i-0341e7dc257e2428c \
  --region us-east-1

# Wait
aws ec2 wait instance-terminated \
  --instance-ids i-0341e7dc257e2428c \
  --region us-east-1

# Relaunch with explicit subnet
aws ec2 run-instances \
  --image-id ami-04680790a315cd58d \
  --instance-type t2.micro \
  --security-group-ids sg-012c205af0c41893a \
  --subnet-id subnet-0c1ddf8bfdedde947 \
  --user-data file:///tmp/xfusion-userdata.sh \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=xfusion-ec2}]' \
  --region us-east-1 \
  --query "Instances[0].InstanceId" \
  --output text

# Deregister old, register new
aws elbv2 deregister-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --targets Id=i-0341e7dc257e2428c \
  --region us-east-1

aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --targets Id=i-0ea7cfa8ef2a7756d,Port=80 \
  --region us-east-1
```

> ***Screenshots: Terminate, wait, and relaunch commands with new instance ID i-0ea7cfa8ef2a7756d returned***
<img width="1037" height="299" alt="image" src="https://github.com/user-attachments/assets/15d241ab-4d2d-452d-a6d3-1a9b9d232cda" />
<img width="1045" height="251" alt="image" src="https://github.com/user-attachments/assets/6927ce22-6910-4eba-8ecb-10f561fe5b39" />
<img width="1034" height="377" alt="image" src="https://github.com/user-attachments/assets/03624216-9eb7-44de-bce8-667456fde013" />

> ***Screenshot: deregister-targets and register-targets commands followed by healthy target health output***
<img width="1028" height="442" alt="image" src="https://github.com/user-attachments/assets/2a40807f-d0f2-4911-a772-852069d058c2" />

**Status:** Resolved. New instance `i-0ea7cfa8ef2a7756d` placed in `us-east-1e`, matching ALB subnet coverage.

---

## Lessons Learned

### 1. Always Specify `--subnet-id` When Launching EC2 for ALB Use

AWS auto-placement selects any available AZ in the region. ALBs only route to targets in AZs covered by their registered subnets. These two selections are independent and will conflict unless the subnet is explicitly specified at launch time.

**Rule:** Before launching any EC2 instance that will serve as an ALB target, identify which AZs the ALB covers. Use `--subnet-id` to pin the instance to one of those AZs. Never rely on auto-placement.

### 2. Audit IAM Permissions Before Deployment, Not During

The `elasticloadbalancing:SetSubnets` permission gap was discovered mid-deployment after the AZ mismatch occurred. A pre-deployment IAM policy review against the required action list would have surfaced this before any resources were created.

**Rule:** Maintain a permissions checklist per deployment type. For ALB + EC2 deployments, the minimum required actions include `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:AuthorizeSecurityGroupIngress`, `elasticloadbalancing:CreateLoadBalancer`, `elasticloadbalancing:CreateTargetGroup`, `elasticloadbalancing:CreateListener`, and `elasticloadbalancing:RegisterTargets`.

### 3. `unused` and `initial` Are Not Failure States

Both states have precise meanings and are part of the normal health check lifecycle. `unused` before listener creation is expected. `initial` after listener creation means health checks have not yet passed the threshold. Neither state requires intervention beyond waiting.

### 4. User Data Script Line Endings Are Silent Killers

The `file` command was unavailable in this environment. Use `cat -A` instead to verify Unix line endings. A script with CRLF (`^M$`) line endings will be uploaded successfully but will fail silently at instance startup with no error surfaced to the user.

---

## Quick Reference

### Pre-Flight Reference Card

```
Account ID      = 683588789756
Region          = us-east-1
DEFAULT_VPC_ID  = vpc-09a4156f3d5e1c2e8
DEFAULT_SG_ID   = sg-03539665823fc6953
SUBNET_ID_1     = subnet-0c1ddf8bfdedde947  (us-east-1e)  <-- ALB + EC2
SUBNET_ID_2     = subnet-0885a17fd0dc2df78  (us-east-1d)  <-- ALB only
UBUNTU_AMI_ID   = ami-04680790a315cd58d
XFUSION_SG_ID   = sg-012c205af0c41893a
INSTANCE_ID     = i-0ea7cfa8ef2a7756d
ALB_ARN         = arn:aws:elasticloadbalancing:us-east-1:683588789756:loadbalancer/app/xfusion-alb/51a4c3075a861f0a
ALB_DNS         = xfusion-alb-1906114081.us-east-1.elb.amazonaws.com
TG_ARN          = arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec
LISTENER_ARN    = arn:aws:elasticloadbalancing:us-east-1:683588789756:listener/app/xfusion-alb/51a4c3075a861f0a/e05591297333655e
```

### Deployment Checklist

- Phase 1: All discovery values recorded
- Phase 2: xfusion-sg created, port 80 from default SG only
- Phase 3: User data script written, Unix line endings confirmed
- Phase 4: EC2 launched into ALB-covered subnet with explicit `--subnet-id`
- Phase 5: ALB created, state = active
- Phase 6: Target group created, instance registered
- Phase 7: Listener created on port 80, forwarding to xfusion-tg
- Phase 8: Default SG allows port 80 inbound from 0.0.0.0/0
- Phase 9: Target health = healthy, curl returns HTTP 200

### Health Check Quick Commands

```bash
# Target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:683588789756:targetgroup/xfusion-tg/38a2c3614324beec \
  --region us-east-1 \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]" \
  --output table

# Nginx via ALB
curl -I http://xfusion-alb-1906114081.us-east-1.elb.amazonaws.com
```

---


<img width="1032" height="469" alt="image" src="https://github.com/user-attachments/assets/ff74a792-caab-4b5b-a298-2eb2a711f250" />





