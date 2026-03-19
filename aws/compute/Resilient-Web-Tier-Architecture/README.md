# Highly Available Web Application on AWS Using Auto Scaling Group and Application Load Balancer

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![EC2](https://img.shields.io/badge/EC2-AutoScaling-blue?logo=amazonaws)
![Nginx](https://img.shields.io/badge/Nginx-1.28.2-green?logo=nginx)
![Status](https://img.shields.io/badge/Status-Production--Ready-brightgreen)
![Region](https://img.shields.io/badge/Region-us--east--1-yellow)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Infrastructure Summary](#infrastructure-summary)
- [Implementation Guide](#implementation-guide)
  - [Phase 1 - Security Group](#phase-1---security-group)
  - [Phase 2 - Amazon Linux 2 AMI](#phase-2---amazon-linux-2-ami)
  - [Phase 3 - Launch Template](#phase-3---launch-template)
  - [Phase 4 - VPC and Subnet Discovery](#phase-4---vpc-and-subnet-discovery)
  - [Phase 5 - Target Group](#phase-5---target-group)
  - [Phase 6 - Application Load Balancer](#phase-6---application-load-balancer)
  - [Phase 7 - ALB Listener](#phase-7---alb-listener)
  - [Phase 8 - Auto Scaling Group](#phase-8---auto-scaling-group)
  - [Phase 9 - CPU-Based Scaling Policy](#phase-9---cpu-based-scaling-policy)
  - [Phase 10 - Health Verification](#phase-10---health-verification)
  - [Phase 11 - End-to-End Validation](#phase-11---end-to-end-validation)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting Reference](#troubleshooting-reference)
- [Resource Summary](#resource-summary)
- [Validation Checklist](#validation-checklist)

---

## Overview

This runbook documents the end-to-end implementation of a **highly available, auto-scaling web application** on AWS using:

- **EC2 Auto Scaling Group (ASG)** to maintain instance health and scale based on CPU utilization
- **Application Load Balancer (ALB)** to distribute incoming HTTP traffic across healthy EC2 instances
- **Nginx** as the web server, installed via the Amazon Linux 2 Extras mechanism

The setup guarantees that a minimum of one EC2 instance is always running, scales to a maximum of two instances when CPU exceeds 50%, and routes all traffic exclusively to healthy targets.

---

## Architecture

```
Internet
    |
    v
[Application Load Balancer] datacenter-alb
    | Port 80 (HTTP)
    v
[Target Group] datacenter-tg
    | Health Check: GET / -> HTTP 200
    v
[Auto Scaling Group] datacenter-asg
    | Min: 1 | Desired: 1 | Max: 2
    | CPU Target Tracking: 50%
    v
[EC2 t2.micro Instances]
    | Amazon Linux 2 | Nginx 1.28.2
    | Security Group: Port 80 open to 0.0.0.0/0
    v
[Default VPC] vpc-0fef15775fee65278 | us-east-1
    | us-east-1f | us-east-1d
```

---

## Problem Statement

The DevOps team requires a production-grade, highly available web application infrastructure on AWS with the following non-negotiable requirements:

- EC2 instances must always be running with automatic self-healing via ASG
- Incoming HTTP traffic must be evenly distributed across all healthy instances
- Instances must automatically scale out when CPU utilization exceeds 50%
- Nginx must be pre-installed and fully operational on every instance at launch via User Data
- All components must reside in the `us-east-1` region

**Resolution:** Implemented a fully automated AWS infrastructure stack using EC2 Launch Templates with encoded User Data bootstrap scripts, an Auto Scaling Group with TargetTracking CPU-based scaling, and an internet-facing Application Load Balancer with ELB health check-based instance routing.

---

## Prerequisites

- AWS CLI installed and configured with valid credentials
- IAM permissions for: `ec2:*`, `elasticloadbalancing:*`, `autoscaling:*`, `cloudwatch:*`
- An active AWS account in `us-east-1` with a default VPC available
- Bash shell with `base64` utility available

---

## Infrastructure Summary

| Resource | Name | Key Configuration |
|---|---|---|
| Security Group | `datacenter-sg` | TCP port 80 inbound from `0.0.0.0/0` |
| Launch Template | `datacenter-launch-template` | Amazon Linux 2, t2.micro, Nginx via amazon-linux-extras |
| Auto Scaling Group | `datacenter-asg` | Min 1, Desired 1, Max 2, ELB health check |
| Scaling Policy | `datacenter-cpu-scaling-policy` | TargetTrackingScaling at 50% CPU |
| Target Group | `datacenter-tg` | HTTP, Port 80, instance target type |
| Load Balancer | `datacenter-alb` | Internet-facing, Application type, Port 80 |
| Region | `us-east-1` | Multi-AZ: us-east-1f, us-east-1d |

---

## Implementation Guide

---

### Phase 1 - Security Group

Create a security group that permits inbound HTTP traffic on port 80 from any IP address, and verify the rule was applied correctly.

**Create the security group:**

```bash
aws ec2 create-security-group \
  --group-name datacenter-sg \
  --description "Security group for Datacenter web servers - allows HTTP on port 80" \
  --region us-east-1
```

**Expected output:**
```json
{
    "GroupId": "sg-03e7ee7f26f9a6441",
    "SecurityGroupArn": "arn:aws:ec2:us-east-1:860767964813:security-group/sg-03e7ee7f26f9a6441"
}
```

> **Screenshot**

<img width="1033" height="699" alt="image" src="https://github.com/user-attachments/assets/a93491d7-2d31-46ce-ae0c-e4507027d9f8" />

> `Terminal showing security group creation output with GroupId highlighted]`

**Add inbound rule for HTTP on port 80:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-name datacenter-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0 \
  --region us-east-1
```

**Expected output:**
```json
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0259b9e24f085f4ab",
            "GroupId": "sg-03e7ee7f26f9a6441",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 80,
            "ToPort": 80,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
```

> **Screenshot**

<img width="1033" height="699" alt="image" src="https://github.com/user-attachments/assets/a93491d7-2d31-46ce-ae0c-e4507027d9f8" />

> `Terminal showing authorize-security-group-ingress output with Return: true and CidrIpv4: 0.0.0.0/0]`

---

### Phase 2 - Amazon Linux 2 AMI

Dynamically retrieve the latest Amazon Linux 2 AMI ID. Hardcoding AMI IDs is a critical anti-pattern as they vary by region and are updated regularly by AWS.

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
    "Name=state,Values=available" \
  --query "Images | sort_by(@, &CreationDate) | [-1].ImageId" \
  --output text \
  --region us-east-1
```

**Expected output:**
```
ami-05024c2628f651b80
```

> **Screenshotr**



> `Terminal showing the describe-images command returning the latest AMI ID]`

---

### Phase 3 - Launch Template

The launch template defines the complete EC2 instance configuration and embeds a base64-encoded User Data script that bootstraps Nginx at instance launch.

> **Critical:** On Amazon Linux 2, Nginx is **not** available in the default `yum` repository. It must be installed using `amazon-linux-extras install nginx1 -y`. Using `yum install -y nginx` will fail silently in User Data with `No package nginx available`.

**Step 3a - Create the User Data bootstrap script:**

```bash
cat << 'EOF' > /tmp/userdata-datacenter.sh
#!/bin/bash
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
EOF
```

**Step 3b - Verify script contents before encoding:**

```bash
cat /tmp/userdata-datacenter.sh
```

**Expected output:**
```bash
#!/bin/bash
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
```

**Step 3c - Base64-encode the User Data script:**

```bash
base64 -w 0 /tmp/userdata-datacenter.sh > /tmp/userdata-datacenter-b64.txt
```

**Step 3d - Create the launch template with all configuration embedded:**

```bash
aws ec2 create-launch-template \
  --launch-template-name datacenter-launch-template \
  --version-description "v1" \
  --launch-template-data "{
    \"ImageId\": \"ami-05024c2628f651b80\",
    \"InstanceType\": \"t2.micro\",
    \"SecurityGroupIds\": [\"sg-03e7ee7f26f9a6441\"],
    \"UserData\": \"$(cat /tmp/userdata-datacenter-b64.txt)\"
  }" \
  --region us-east-1
```

**Expected output:**
```json
{
    "LaunchTemplate": {
        "LaunchTemplateId": "lt-044b18f242a5d9e4d",
        "LaunchTemplateName": "datacenter-launch-template",
        "DefaultVersionNumber": 1,
        "LatestVersionNumber": 1
    }
}
```

**Step 3e - Verify the launch template exists:**

```bash
aws ec2 describe-launch-templates \
  --launch-template-names datacenter-launch-template \
  --region us-east-1
```

> **Screenshot**



> `Terminal showing launch template creation output with LaunchTemplateId lt-044b18f242a5d9e4d and version numbers]`

---

### Phase 4 - VPC and Subnet Discovery

Retrieve the default VPC ID and enumerate all available subnets. The ALB requires subnets in a minimum of two distinct Availability Zones.

**Get the default VPC ID:**

```bash
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text \
  --region us-east-1
```

**Expected output:**
```
vpc-0fef15775fee65278
```

**List all subnets in the VPC:**

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0fef15775fee65278" \
  --query "Subnets[*].[SubnetId,AvailabilityZone]" \
  --output table \
  --region us-east-1
```

**Expected output:**
```
--------------------------------------------
|              DescribeSubnets             |
+---------------------------+--------------+
|  subnet-08fc950d95b191d64 |  us-east-1f  |
|  subnet-0139711d961258ad5 |  us-east-1d  |
|  subnet-0a15f1efa3c2fcd38 |  us-east-1c  |
|  subnet-0b43d450decb84c83 |  us-east-1b  |
|  subnet-079db625185781d05 |  us-east-1e  |
|  subnet-074540c3a5f02efd8 |  us-east-1a  |
+---------------------------+--------------+
```

> **Screenshot**



> `Terminal showing VPC ID output and full subnet table with all six Availability Zones]`

**Selected subnets for this deployment:**

| Subnet ID | Availability Zone | Purpose |
|---|---|---|
| `subnet-08fc950d95b191d64` | us-east-1f | ALB + ASG AZ 1 |
| `subnet-0139711d961258ad5` | us-east-1d | ALB + ASG AZ 2 |

---

### Phase 5 - Target Group

Create the target group that the ALB uses to route requests and execute HTTP health checks against registered instances.

```bash
aws elbv2 create-target-group \
  --name datacenter-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0fef15775fee65278 \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --target-type instance \
  --region us-east-1
```

**Expected output (key fields):**
```json
{
    "TargetGroups": [
        {
            "TargetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:860767964813:targetgroup/datacenter-tg/fc4a7af9743287ef",
            "TargetGroupName": "datacenter-tg",
            "Protocol": "HTTP",
            "Port": 80,
            "HealthCheckPath": "/",
            "HealthyThresholdCount": 2,
            "UnhealthyThresholdCount": 2,
            "Matcher": { "HttpCode": "200" },
            "TargetType": "instance"
        }
    ]
}
```

> **Screenshot**



> `Terminal showing create-target-group output with TargetGroupArn and health check configuration]`

---

### Phase 6 - Application Load Balancer

Create the internet-facing Application Load Balancer spanning two Availability Zones.

```bash
aws elbv2 create-load-balancer \
  --name datacenter-alb \
  --subnets subnet-08fc950d95b191d64 subnet-0139711d961258ad5 \
  --security-groups sg-03e7ee7f26f9a6441 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --region us-east-1
```

**Expected output (key fields):**
```json
{
    "LoadBalancers": [
        {
            "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-east-1:860767964813:loadbalancer/app/datacenter-alb/fe7f0a42f28ac09f",
            "DNSName": "datacenter-alb-910242966.us-east-1.elb.amazonaws.com",
            "Scheme": "internet-facing",
            "Type": "application",
            "State": { "Code": "provisioning" }
        }
    ]
}
```

> **Note:** State `provisioning` is expected and normal. The ALB becomes `active` within 1 to 2 minutes.

> **Screenshot**



> `Terminal showing create-load-balancer output with LoadBalancerArn and DNSName]`

---

### Phase 7 - ALB Listener

Create a listener on port 80 that forwards all incoming HTTP traffic to the target group.

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:860767964813:loadbalancer/app/datacenter-alb/fe7f0a42f28ac09f \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:860767964813:targetgroup/datacenter-tg/fc4a7af9743287ef \
  --region us-east-1
```

**Expected output (key fields):**
```json
{
    "Listeners": [
        {
            "Port": 80,
            "Protocol": "HTTP",
            "DefaultActions": [
                {
                    "Type": "forward",
                    "TargetGroupArn": "arn:aws:elasticloadbalancing:us-east-1:860767964813:targetgroup/datacenter-tg/fc4a7af9743287ef"
                }
            ]
        }
    ]
}
```

> **Screenshot**



> `Terminal showing create-listener output confirming Port 80, Protocol HTTP, and forward action to datacenter-tg]`

---

### Phase 8 - Auto Scaling Group

Create the Auto Scaling Group referencing the launch template. The `$Latest` version reference ensures new instances always use the most current template version.

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name datacenter-asg \
  --launch-template "LaunchTemplateName=datacenter-launch-template,Version=\$Latest" \
  --min-size 1 \
  --max-size 2 \
  --desired-capacity 1 \
  --target-group-arns arn:aws:elasticloadbalancing:us-east-1:860767964813:targetgroup/datacenter-tg/fc4a7af9743287ef \
  --vpc-zone-identifier "subnet-08fc950d95b191d64,subnet-0139711d961258ad5" \
  --health-check-type ELB \
  --health-check-grace-period 120 \
  --region us-east-1
```

> **Note:** A silent exit with no output and exit code 0 confirms successful creation.

**Verify ASG configuration:**

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names datacenter-asg \
  --region us-east-1
```

> **Screenshot**



> `Terminal showing describe-auto-scaling-groups output with MinSize: 1, MaxSize: 2, DesiredCapacity: 1, HealthCheckType: ELB, and TargetGroupARNs]`

---

### Phase 9 - CPU-Based Scaling Policy

Attach a TargetTracking scaling policy that maintains average CPU utilization at 50%. AWS automatically provisions both scale-out (AlarmHigh) and scale-in (AlarmLow) CloudWatch alarms.

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name datacenter-asg \
  --policy-name datacenter-cpu-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration "{
    \"PredefinedMetricSpecification\": {
      \"PredefinedMetricType\": \"ASGAverageCPUUtilization\"
    },
    \"TargetValue\": 50.0
  }" \
  --region us-east-1
```

**Expected output:**
```json
{
    "PolicyARN": "arn:aws:autoscaling:us-east-1:860767964813:scalingPolicy:47a13cf8-...",
    "Alarms": [
        {
            "AlarmName": "TargetTracking-datacenter-asg-AlarmHigh-3dfe231c-..."
        },
        {
            "AlarmName": "TargetTracking-datacenter-asg-AlarmLow-2ce25787-..."
        }
    ]
}
```

> **Screenshot**



> `Terminal showing put-scaling-policy output with PolicyARN and both AlarmHigh and AlarmLow CloudWatch alarms created]`

---

### Phase 10 - Health Verification

Wait approximately 2 minutes after ASG creation for the EC2 instance to launch and the User Data bootstrap script to complete execution.

**Check ASG instance lifecycle state:**

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names datacenter-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table \
  --region us-east-1
```

**Expected output:**
```
-----------------------------------------------------
|             DescribeAutoScalingGroups             |
+---------------------+---------------+-------------+
|  i-015595327795633fe|  InService    |  Healthy    |
+---------------------+---------------+-------------+
```

**Check ALB target group health:**

```bash
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:860767964813:targetgroup/datacenter-tg/fc4a7af9743287ef \
  --region us-east-1
```

**Expected output:**
```json
{
    "TargetHealthDescriptions": [
        {
            "Target": {
                "Id": "i-015595327795633fe",
                "Port": 80
            },
            "HealthCheckPort": "80",
            "TargetHealth": {
                "State": "healthy"
            }
        }
    ]
}
```

> **Screenshot**



> `Terminal showing ASG instance table with InService/Healthy AND describe-target-health output with State: healthy]`

---

### Phase 11 - End-to-End Validation

Confirm the full traffic path from public internet through the ALB to the EC2 instance returns the Nginx welcome page.

```bash
curl -v http://datacenter-alb-910242966.us-east-1.elb.amazonaws.com
```

**Expected output (key indicators):**
```
* Connected to datacenter-alb-910242966.us-east-1.elb.amazonaws.com port 80
< HTTP/1.1 200 OK
< Server: nginx/1.28.2
...
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and working.</p>
```

> **Screenshot**



> `Terminal showing full curl -v output with HTTP/1.1 200 OK, Server: nginx/1.28.2 header, and Welcome to nginx! in the HTML body]`

> **Screenshot**



> `Browser tab opened to the ALB DNS URL displaying the default Nginx welcome page]`

---

## Best Practices

### Security Hardening

- **Principle of Least Privilege:** The security group exposes only TCP port 80. No SSH port 22 is opened to the internet. Use AWS Systems Manager Session Manager for all administrative instance access.
- **Production pattern:** Create a dedicated security group for the ALB and restrict the instance security group to accept inbound traffic exclusively from the ALB security group rather than from `0.0.0.0/0`.
- **HTTPS in production:** Replace the HTTP listener with an HTTPS listener using an ACM certificate and redirect HTTP to HTTPS via a listener rule.

### Launch Template Design

- **Use `$Latest` version** in ASG configuration so new instances always inherit the most recent template changes without requiring ASG updates.
- **Never hardcode AMI IDs.** Query the latest AMI dynamically at provisioning time using `describe-images` with `sort_by` on `CreationDate`.
- **Validate User Data before encoding.** Review the script carefully, then verify the encoded value decodes correctly before applying it to production templates.
- **On Amazon Linux 2, always use `amazon-linux-extras`** for packages outside the default repository including `nginx1`, `docker`, `python3.8`, and `ruby3.0`.

### Auto Scaling Configuration

- **Set `health-check-grace-period` to at least 120 seconds** to allow the full User Data bootstrap (package installation) to complete before health check evaluation begins.
- **Always use `ELB` health check type** (not the default `EC2`) on the ASG so the group replaces instances that fail ALB health checks, catching application-level failures that EC2 status checks cannot detect.
- **TargetTrackingScaling** is the preferred policy type for CPU-based scaling. AWS handles all scaling arithmetic automatically and provisions the required CloudWatch alarms.
- **Multi-AZ is mandatory for production.** Distribute the ASG and ALB across a minimum of two Availability Zones to tolerate AZ-level failures.

### Operational Readiness

- **Tag every resource** with `Environment`, `Project`, `Owner`, and `CostCenter` tags for cost allocation, IAM policy enforcement, and inventory reporting.
- **Enable ALB access logs** to an S3 bucket for audit trails, traffic pattern analysis, and post-incident forensics.
- **Monitor these CloudWatch metrics at minimum:**
  - ALB: `HealthyHostCount`, `RequestCount`, `TargetResponseTime`, `HTTPCode_ELB_5XX_Count`
  - ASG: `GroupInServiceInstances`, `GroupDesiredCapacity`
  - EC2: `CPUUtilization`
- **Set up CloudWatch alarms** on `HealthyHostCount < 1` with SNS notification for immediate on-call alerting.

---

## Lessons Learned

### Amazon Linux 2 Package Management

**Problem:** `yum install -y nginx` in the User Data script silently fails on Amazon Linux 2 instances with the error `No package nginx available`. The EC2 instance launches successfully and appears healthy from the ASG perspective, but Nginx is not installed and port 80 is not listening, causing ALB health checks to fail.

**Root Cause:** Amazon Linux 2 does not include Nginx in its default `yum` repositories. Nginx is distributed exclusively as an Amazon Linux Extras topic named `nginx1`.

**Resolution:**

```bash
# WRONG - fails on Amazon Linux 2
yum install -y nginx

# CORRECT - required for Amazon Linux 2
amazon-linux-extras install nginx1 -y
```

**Scope:** This applies to all Amazon Linux 2 instances. Amazon Linux 2023 uses `dnf` and includes Nginx directly, so this issue does not affect AL2023.

---

### User Data Bootstrap Timing

**Observation:** The ASG may report an instance as `InService` and `Healthy` from an EC2 status perspective before `amazon-linux-extras install nginx1 -y` has finished executing. Package installation takes 60 to 120 seconds on a cold t2.micro instance. ALB health checks will report `initial` or `unhealthy` during this window.

**Resolution:** Set `--health-check-grace-period 120` on the ASG. For heavier bootstrapping workloads involving multiple packages, configuration management, or application deployment, increase this value to 180 to 300 seconds accordingly.

---

### Dynamic AMI Resolution

**Observation:** Hardcoded AMI IDs become stale. AWS publishes updated Amazon Linux 2 AMIs regularly with kernel patches and security updates. Using a stale AMI can introduce known vulnerabilities or miss critical patches.

**Resolution:** Always resolve the AMI ID dynamically at infrastructure provisioning time:

```bash
aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
    "Name=state,Values=available" \
  --query "Images | sort_by(@, &CreationDate) | [-1].ImageId" \
  --output text \
  --region us-east-1
```

---

### ALB Multi-AZ Requirement

**Observation:** AWS requires that an Application Load Balancer be associated with subnets in at least two Availability Zones. Attempting to create an ALB with a single subnet will return a client error.

**Resolution:** Always query available subnets in the target VPC and explicitly select subnets from two or more distinct Availability Zones before issuing the `create-load-balancer` call.

---

## Troubleshooting Reference

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `No package nginx available` in User Data | Nginx not in default AL2 yum repo | Replace `yum install -y nginx` with `amazon-linux-extras install nginx1 -y` |
| Target health state: `initial` | User Data script still running | Wait 90 to 120 seconds; recheck with `describe-target-health` |
| Target health state: `unhealthy` | Nginx not started or port 80 not listening | Inspect console output via `aws ec2 get-console-output` |
| `curl: (7) Connection refused` on instance IP | Nginx not running on the instance | Verify User Data ran successfully; check console logs |
| ASG `Instances` array is empty | Instance is still launching | Wait 60 seconds; run `describe-auto-scaling-groups` again |
| ALB returns HTTP 502 Bad Gateway | All registered targets are unhealthy | Check target group health; verify Nginx is running on each instance |
| ALB DNS does not resolve | ALB still provisioning | Wait 1 to 2 minutes for the ALB state to change from `provisioning` to `active` |

---

## Resource Summary

```
Account ID           : 860767964813
Region               : us-east-1

Security Group       : sg-03e7ee7f26f9a6441
                       (datacenter-sg)

Launch Template      : lt-044b18f242a5d9e4d
                       (datacenter-launch-template v1)

Target Group ARN     : arn:aws:elasticloadbalancing:us-east-1:
                       860767964813:targetgroup/datacenter-tg/fc4a7af9743287ef

Load Balancer ARN    : arn:aws:elasticloadbalancing:us-east-1:
                       860767964813:loadbalancer/app/datacenter-alb/fe7f0a42f28ac09f

ALB DNS Name         : datacenter-alb-910242966.us-east-1.elb.amazonaws.com

Auto Scaling Group   : datacenter-asg
Scaling Policy       : datacenter-cpu-scaling-policy
                       (TargetTrackingScaling | 50% ASGAverageCPUUtilization)

VPC                  : vpc-0fef15775fee65278
Subnets              : subnet-08fc950d95b191d64 (us-east-1f)
                       subnet-0139711d961258ad5 (us-east-1d)
```

---

## Validation Checklist

✅ Security group `datacenter-sg` created with TCP 80 inbound from `0.0.0.0/0`

✅ Inbound rule confirmed with `Return: true` in CLI output

✅ Latest Amazon Linux 2 AMI ID retrieved dynamically

✅ User Data script verified to use `amazon-linux-extras install nginx1 -y`

✅ User Data base64-encoded and embedded in launch template

✅ Launch template `datacenter-launch-template` exists with `LatestVersionNumber: 1`

✅ Default VPC and at least two subnets across different AZs identified

✅ Target group `datacenter-tg` created with HTTP health check on path `/` returning 200

✅ ALB `datacenter-alb` created as internet-facing across two AZs

✅ ALB listener configured on port 80 forwarding to `datacenter-tg`

✅ ASG `datacenter-asg` created with Min 1, Desired 1, Max 2

✅ ASG `health-check-type` set to `ELB` with `health-check-grace-period` of 120 seconds

✅ CPU scaling policy attached targeting 50% utilization with both CloudWatch alarms created

✅ ASG instance shows `LifecycleState: InService` and `HealthStatus: Healthy`

✅ Target group shows `State: healthy` for registered instance

✅ `curl http://datacenter-alb-910242966.us-east-1.elb.amazonaws.com` returns HTTP 200 with `Welcome to nginx!`

---



<img width="1029" height="678" alt="image" src="https://github.com/user-attachments/assets/f592799f-05e8-49cf-bbc3-322b72081b39" />
<img width="1030" height="859" alt="image" src="https://github.com/user-attachments/assets/b14e89b3-ac4f-44c9-aeec-387d5bb8a0a0" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/7901f4a4-4d59-4d7a-bcba-74b93e2f4ffa" />
<img width="1031" height="529" alt="image" src="https://github.com/user-attachments/assets/2c672e2e-ed54-4895-8576-9380c4aa36ca" />
<img width="1034" height="838" alt="image" src="https://github.com/user-attachments/assets/f9b1cbf7-f3dd-471d-8ab9-bccdd9986ab5" />
<img width="1034" height="874" alt="image" src="https://github.com/user-attachments/assets/f8d8c179-7ab4-4516-8bf7-bbe060766ec0" />
<img width="1029" height="860" alt="image" src="https://github.com/user-attachments/assets/de7854cb-34e4-4e44-9bdb-0817e0ae8a5d" />
<img width="1034" height="785" alt="image" src="https://github.com/user-attachments/assets/c68028f8-c43e-4012-a313-5d9829614805" />
<img width="1030" height="825" alt="image" src="https://github.com/user-attachments/assets/f66fcf0e-ae39-4342-89d6-8a361ec58d14" />
<img width="1032" height="747" alt="image" src="https://github.com/user-attachments/assets/39d15344-536f-4ea9-8bee-98b5783ddf88" />
<img width="1029" height="855" alt="image" src="https://github.com/user-attachments/assets/4f03dd84-72e6-4c9c-8acb-1bd978cb7521" />
