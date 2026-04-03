# AWS Application Load Balancer Web Ingress: EC2 + Nginx End-to-End Traffic Routing

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Environment Details](#environment-details)
- [Implementation](#implementation)
  - [Step 1: Discover Existing EC2 Instance](#step-1-discover-existing-ec2-instance)
  - [Step 2: Confirm Default VPC](#step-2-confirm-default-vpc)
  - [Step 3: Create Security Group for ALB](#step-3-create-security-group-for-alb)
  - [Step 4: Create Target Group](#step-4-create-target-group)
  - [Step 5: Register EC2 Instance with Target Group](#step-5-register-ec2-instance-with-target-group)
  - [Step 6: Retrieve Subnets for ALB](#step-6-retrieve-subnets-for-alb)
  - [Step 7: Create Application Load Balancer](#step-7-create-application-load-balancer)
  - [Step 8: Create HTTP Listener](#step-8-create-http-listener)
  - [Step 9: Allow ALB Traffic to EC2](#step-9-allow-alb-traffic-to-ec2)
- [Validation](#validation)
- [Validation Checklist](#validation-checklist)
- [Outcome](#outcome)
- [Key Learnings](#key-learnings)
- [Operational Considerations](#operational-considerations)

---

## Project Overview

This document details the deployment of an **internet-facing AWS Application Load Balancer (ALB)** that routes HTTP traffic to an existing **EC2 instance running Nginx** in the **us-east-1** region. The objective was to validate **end-to-end Layer 7 traffic flow** from the public internet through the ALB to the backend EC2 instance using **target groups and security group isolation**.

This setup represents a foundational **web ingress architecture** commonly used in scalable, production-grade cloud environments and serves as the basis for more advanced patterns such as Auto Scaling, HTTPS termination, and path-based routing.

---

## Problem Statement

Direct public exposure of EC2 instances creates significant security and operational risks:

- **No traffic distribution:** A single instance exposed directly becomes a single point of failure.
- **Uncontrolled ingress:** Security groups on the instance must allow traffic from the entire internet, increasing the attack surface.
- **Limited scalability:** Without a load balancer, horizontal scaling requires DNS-level intervention and lacks seamless traffic shifting.
- **No Layer 7 visibility:** Direct EC2 exposure provides no native HTTP-level routing, health checking, or request inspection.

**Solution:** Deploy an Application Load Balancer as the single public entry point, enforcing security group isolation so the EC2 instance only accepts traffic originating from the ALB. This decouples ingress from compute, enables health-based routing, and positions the architecture for future scaling.

---

## Architecture

```
             ┌────────────────────────┐
             │  Application Load      │
             │  Balancer (ALB)        │
             │  HTTP :80              │
             └───────────┬────────────┘
                         │
               ┌─────────▼─────────┐
               │ Target Group      │
               │ HTTP :80          │
               └─────────┬─────────┘
                         │
                ┌────────▼────────┐
                │ EC2 Instance    │
                │ devops-ec2      │
                │ Nginx :80       │
                └─────────────────┘
```

**Traffic Flow:**
1. Client sends HTTP request to the ALB DNS endpoint.
2. ALB listener on port 80 receives the request.
3. Listener forwards the request to the `devops-tg` target group.
4. Target group routes the request to the registered EC2 instance on port 80.
5. Nginx processes and responds to the request.
6. Response is returned to the client through the ALB.

---

## Technologies Used

- **Amazon EC2** - Backend compute running Nginx
- **Application Load Balancer (ALB)** - Layer 7 internet-facing ingress
- **Target Groups** - Logical grouping of backend targets with health checking
- **Security Groups** - Stateful firewall rules for ingress isolation
- **AWS CLI** - Infrastructure provisioning and verification
- **Nginx** - Web server on the backend EC2 instance

---

## Environment Details

| Component | Value |
|---|---|
| Region | us-east-1 |
| VPC | Default VPC |
| EC2 Instance Name | devops-ec2 |
| Instance Type | t2.micro |
| Load Balancer | devops-alb |
| Target Group | devops-tg |
| ALB Security Group | devops-sg |

---

## Implementation

### Step 1: Discover Existing EC2 Instance

Before creating any load balancer resources, identify and confirm the details of the backend EC2 instance. This ensures that all subsequent configuration references the correct instance ID, VPC, and subnet.

```bash
aws ec2 describe-instances
```

Key values extracted from the output:

```
Instance ID:    i-011ae72a16b4398f6
VPC ID:         vpc-0c3f56dfe99c83d0d
Subnet:         subnet-00b5049732d6fd4eb
Security Group: default
Public IP:      54.172.87.251
Instance Type:  t2.micro
State:          running
AZ:             us-east-1b
```

> **Operational Note:** Always confirm the instance is in a `running` state and is reachable on port 80 before registering it with a target group. A stopped or misconfigured instance will immediately fail health checks.

**Screenshots: EC2 instance describe output showing instance metadata and network configuration**

<img width="1039" height="805" alt="image" src="https://github.com/user-attachments/assets/9db485cd-3b2d-4b09-b3b9-d049498a51fb" />
<img width="1034" height="828" alt="image" src="https://github.com/user-attachments/assets/a96ff6d9-c1e7-4ef8-80cf-5222400f45b8" />
<img width="1033" height="829" alt="image" src="https://github.com/user-attachments/assets/e9412cb5-c775-4336-9de0-ca9f7f66fcde" />
<img width="1035" height="870" alt="image" src="https://github.com/user-attachments/assets/fb14a97a-917f-4b14-bd57-19e73b2ff7dc" />

>**EC2 instance confirm running state, Instance ID, VPC, Subnet, and IP address details**

---

### Step 2: Confirm Default VPC

Confirm that the existing EC2 instance resides in the **default VPC** and retrieve its VPC ID. This value is required for subsequent resource provisioning to ensure all components are deployed within the same network boundary.

```bash
aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

Expected output:

```
vpc-0c3f56dfe99c83d0d
```

> **Operational Note:** If operating in a non-default VPC or a multi-VPC environment, ensure that the ALB subnets, target groups, and security groups all reside in the same VPC. Cross-VPC routing is not supported without explicit VPC peering or Transit Gateway configuration.

**Screenshot: Default VPC ID retrieval confirming vpc-0c3f56dfe99c83d0d**

<img width="1041" height="368" alt="image" src="https://github.com/user-attachments/assets/1ae5db7b-3b6b-4f4d-8404-93d83728d391" />

---

### Step 3: Create Security Group for ALB

Create a **dedicated security group** for the ALB that allows inbound HTTP traffic from the public internet. Separating the ALB security group from the EC2 security group enables precise security group chaining in a later step.

**Create the security group:**

```bash
aws ec2 create-security-group \
  --group-name devops-sg \
  --description "Security group for devops-alb" \
  --vpc-id vpc-0c3f56dfe99c83d0d
```

Output:

```json
{
    "GroupId": "sg-0f2b9556d703a6a3c",
    "SecurityGroupArn": "arn:aws:ec2:us-east-1:605993631845:security-group/sg-0f2b9556d703a6a3c"
}
```

**Authorize inbound HTTP access on port 80 from all sources:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-name devops-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

> **Security Note:** Allowing `0.0.0.0/0` is appropriate for the ALB because it is the intended public entry point. The EC2 instance security group will be restricted to accept traffic only from this ALB security group, effectively preventing any direct internet access to the backend.

> **Risk:** Ensure that the EC2 instance's security group does **not** independently allow `0.0.0.0/0` on port 80 after this configuration, as that would bypass the ALB entirely.

**Screenshot: ALB security group creation and inbound HTTP rule authorization showing GroupId sg-0f2b9556d703a6a3c and CidrIpv4 0.0.0.0/0**

![Terminal showing create-security-group and authorize-security-group-ingress commands with successful SecurityGroupRules response](https://github.com/user-attachments/assets/fb14a97a-917f-4b14-bd57-19e73b2ff7dc)

---

### Step 4: Create Target Group

A **target group** defines the destination backend for ALB-forwarded traffic. It also manages health checks that determine whether the EC2 instance is eligible to receive requests.

```bash
aws elbv2 create-target-group \
  --name devops-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0c3f56dfe99c83d0d \
  --target-type instance
```

Key configuration values confirmed in the output:

```
TargetGroupArn:              arn:aws:elasticloadbalancing:us-east-1:605993631845:targetgroup/devops-tg/ec83614d03aa814d
HealthCheckProtocol:         HTTP
HealthCheckPort:             traffic-port
HealthCheckIntervalSeconds:  30
HealthCheckTimeoutSeconds:   5
HealthyThresholdCount:       5
UnhealthyThresholdCount:     2
HealthCheckPath:             /
Matcher HttpCode:            200
TargetType:                  instance
```

> **Operational Note:** The default health check path is `/` and expects an HTTP `200` response. If the Nginx configuration serves a non-200 response on `/` (e.g., a redirect), update `--health-check-path` or `--matcher` accordingly to prevent the instance from being marked unhealthy.

> **Edge Case:** `HealthCheckIntervalSeconds: 30` with `HealthyThresholdCount: 5` means it takes up to **2.5 minutes** for a newly registered instance to become healthy. Account for this delay during deployments.

**Screenshot: Target group creation output confirming devops-tg with HTTP health check configuration**

![Terminal showing aws elbv2 create-target-group command response with TargetGroupArn, HealthCheck settings, and TargetType instance](https://github.com/user-attachments/assets/e72780dc-56fa-48f8-9a0a-7308b2d2faee)

---

### Step 5: Register EC2 Instance with Target Group

Register the EC2 instance as a target within the `devops-tg` target group. Once registered, the ALB will begin routing health checks to the instance.

```bash
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:605993631845:targetgroup/devops-tg/ec83614d03aa814d \
  --targets Id=i-011ae72a16b4398f6
```

> **Operational Note:** A successful `register-targets` call returns no output. Verify registration status by running `aws elbv2 describe-target-health --target-group-arn <ARN>`. The target will transition from `initial` to `healthy` once health checks pass.

> **Troubleshooting:** If the target remains `unhealthy`, confirm that:
> - Nginx is active and listening on port 80.
> - The EC2 security group allows inbound TCP on port 80 (added in Step 9).
> - The health check path returns an HTTP 200 response.

**Screenshot: Target group creation and EC2 instance registration command with full target group ARN**

![Terminal showing create-target-group output followed by register-targets command targeting instance i-011ae72a16b4398f6](https://github.com/user-attachments/assets/4782d0f9-177e-4c39-9f97-eb559ea407d2)

---

### Step 6: Retrieve Subnets for ALB

An ALB requires **a minimum of two subnets in different Availability Zones** to ensure high availability. Retrieve all available subnets in the default VPC.

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0c3f56dfe99c83d0d" \
  --query "Subnets[*].SubnetId" \
  --output text
```

Subnets returned:

```
subnet-09d0365261e4c44c6    (us-east-1e)
subnet-00b5049732d6fd4eb    (us-east-1b)
subnet-0f9af1437dc5cdccb
subnet-04828bd2be0e392c6
subnet-08a0a615da76be77d
subnet-0933db13b5221372c
```

Two subnets were selected for ALB deployment to span multiple Availability Zones:

- `subnet-09d0365261e4c44c6` (us-east-1e)
- `subnet-00b5049732d6fd4eb` (us-east-1b)

> **Best Practice:** Always select subnets in **at least two distinct Availability Zones** for ALB deployment. This ensures the load balancer remains reachable even if an entire AZ experiences an outage.

**Screenshot: All subnet IDs in the default VPC returned for ALB subnet selection**

![Terminal showing describe-subnets output listing six SubnetIds across multiple Availability Zones](https://github.com/user-attachments/assets/6802a510-d17f-4f87-ad93-eb2c49e6a645)

---

### Step 7: Create Application Load Balancer

Create the internet-facing ALB, attaching it to the two selected subnets and associating the `devops-sg` security group.

```bash
aws elbv2 create-load-balancer \
  --name devops-alb \
  --subnets subnet-09d0365261e4c44c6 subnet-00b5049732d6fd4eb \
  --security-groups sg-0f2b9556d703a6a3c
```

Key values confirmed in the output:

```
LoadBalancerArn:  arn:aws:elasticloadbalancing:us-east-1:605993631845:loadbalancer/app/devops-alb/6c8555f48c0d8be7
DNSName:          devops-alb-1274082417.us-east-1.elb.amazonaws.com
Scheme:           internet-facing
State:            provisioning
Type:             application
AvailabilityZones:
  - us-east-1b (subnet-00b5049732d6fd4eb)
  - us-east-1e (subnet-09d0365261e4c44c6)
```

> **Operational Note:** The ALB enters a `provisioning` state immediately after creation. It transitions to `active` within 1 to 3 minutes. Do not proceed to listener creation or testing until the ALB state is `active`. Verify with:
> ```bash
> aws elbv2 describe-load-balancers --names devops-alb --query 'LoadBalancers[0].State'
> ```

> **Important:** The `DNSName` returned here is the **public entry point** for all traffic. This DNS name should be used in DNS records (e.g., CNAME or Route 53 alias) for any custom domain configuration.

**Screenshot: ALB creation output confirming internet-facing scheme, DNS name, availability zones, and provisioning state**

![Terminal showing aws elbv2 create-load-balancer response with LoadBalancerArn, DNSName, Scheme internet-facing, and AvailabilityZones](https://github.com/user-attachments/assets/0589ef65-e44c-4742-8ec8-a196f8be51a1)

---

### Step 8: Create HTTP Listener

A **listener** instructs the ALB on how to handle incoming requests on a specific port and protocol. Configure the listener to forward all HTTP traffic on port 80 to the `devops-tg` target group.

```bash
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:605993631845:loadbalancer/app/devops-alb/6c8555f48c0d8be7 \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:605993631845:targetgroup/devops-tg/ec83614d03aa814d
```

Key values confirmed in the output:

```
ListenerArn:     arn:aws:elasticloadbalancing:us-east-1:605993631845:listener/app/devops-alb/6c8555f48c0d8be7/fa861150f36f9bf7
Port:            80
Protocol:        HTTP
DefaultActions:  Type=forward -> devops-tg
```

> **Best Practice:** For production environments, HTTP listeners should be configured with a **redirect action** (301) to HTTPS rather than a forward. This ensures all traffic is encrypted in transit. Add an HTTPS listener with an ACM certificate after confirming the HTTP flow is functional.

> **Edge Case:** If multiple target groups are required (e.g., for path-based or host-based routing), listener rules with conditions can be added after the default listener is in place.

**Screenshot: HTTP listener creation output confirming port 80 forward action to devops-tg target group**

![Terminal showing aws elbv2 create-listener response with ListenerArn, Protocol HTTP, Port 80, and DefaultActions Type forward](https://github.com/user-attachments/assets/0212a8f0-806b-4eba-80d1-8834d412a0f8)

---

### Step 9: Allow ALB Traffic to EC2

Update the **EC2 instance's security group** to accept inbound TCP traffic on port 80, but **exclusively from the ALB security group**. This is the critical step that enforces security group isolation and prevents direct internet access to the backend.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0d5e8ddf23e6d5ab3 \
  --protocol tcp \
  --port 80 \
  --source-group sg-0f2b9556d703a6a3c
```

This rule translates to:

```
EC2 Security Group (sg-0d5e8ddf23e6d5ab3):
  Inbound: TCP :80  Source: sg-0f2b9556d703a6a3c (devops-sg / ALB only)
```

> **Security Note:** Using a **source security group** instead of a CIDR range (e.g., `0.0.0.0/0`) is a security best practice. It ensures that only resources associated with the ALB security group can initiate connections to the EC2 instance on port 80, regardless of their IP address. This is the correct pattern for production security group chaining.

> **Troubleshooting:** If the target health check remains unhealthy after this step, verify the rule was applied to the correct security group ID associated with the EC2 instance (not the ALB group).

**Screenshot: EC2 security group ingress rule added with ALB security group as the source, confirming ReferencedGroupInfo**

![Terminal showing authorize-security-group-ingress command with source-group sg-0f2b9556d703a6a3c and SecurityGroupRules response with ReferencedGroupInfo](https://github.com/user-attachments/assets/605bc0a2-db1b-44d3-a6f6-db88a4e40e5a)

---

## Validation

Retrieve the ALB DNS name and confirm the application is reachable via HTTP.

**Retrieve ALB DNS Name:**

```bash
aws elbv2 describe-load-balancers \
  --names devops-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```

Expected output:

```
devops-alb-1274082417.us-east-1.elb.amazonaws.com
```

**Test HTTP access:**

```bash
curl -I http://devops-alb-1274082417.us-east-1.elb.amazonaws.com
```

A successful response returns an HTTP `200 OK` from the Nginx backend, confirming end-to-end Layer 7 traffic flow through the ALB.

**Screenshot: ALB DNS name retrieved and end-to-end HTTP traffic confirmed via the public ALB endpoint**

![Terminal showing aws elbv2 describe-load-balancers command returning devops-alb-1274082417.us-east-1.elb.amazonaws.com](https://github.com/user-attachments/assets/38b8cdc3-dbf3-45b5-98c0-a909ca30eb99)

---

## Validation Checklist

| Check | Status |
|---|---|
| EC2 instance running | PASS |
| ALB created and active | PASS |
| Target group configured | PASS |
| EC2 registered as target | PASS |
| Listener forwarding HTTP :80 traffic | PASS |
| Security group isolation enforced | PASS |
| Application reachable via ALB DNS | PASS |

---

## Outcome

- Successfully implemented Layer 7 web ingress using an AWS Application Load Balancer.
- Public HTTP traffic is routed securely through the ALB to the Nginx backend.
- The EC2 instance is protected from direct internet exposure through security group chaining.
- The ALB DNS endpoint serves as the canonical public entry point.
- The architecture is ready for the following enhancements:
  - **Auto Scaling Group** integration for horizontal scaling
  - **HTTPS listener** with ACM certificate for TLS termination
  - **Path-based or host-based routing** using listener rules
  - **WAF integration** for Layer 7 threat protection

---

## Key Learnings

- **ALB operates at Layer 7 (HTTP/HTTPS):** It can inspect HTTP headers, paths, and hostnames, enabling intelligent routing decisions that are not possible at Layer 4.
- **Target groups decouple ingress from compute:** Registering and deregistering instances from a target group enables zero-downtime deployments and seamless scaling without changing listener configuration.
- **Security group chaining enforces isolation:** Referencing a security group as the source rule (rather than a CIDR) dynamically restricts access to only the intended AWS resource, regardless of IP changes.
- **ALB DNS endpoint is the public entry point:** The ALB DNS name should be used as the base for all public-facing DNS records. Avoid referencing EC2 public IPs directly.
- **Multi-subnet ALB enables high availability:** Spanning the ALB across two or more Availability Zones ensures the load balancer itself is not a single point of failure.
- **Health check tuning is critical:** Default health check settings may be too conservative for fast-startup services. Adjust `HealthCheckIntervalSeconds` and `HealthyThresholdCount` to match the application's startup profile.

---

## Operational Considerations

**Cost Awareness:**
- ALBs incur an hourly charge plus LCU (Load Balancer Capacity Unit) charges based on traffic volume. Terminate unused ALBs to avoid unnecessary spend.

**HTTPS Readiness:**
- This deployment uses HTTP only. For any production or regulated environment, add an HTTPS listener using an AWS Certificate Manager (ACM) certificate and redirect all HTTP traffic to HTTPS.

**Logging and Observability:**
- Enable ALB **access logs** to an S3 bucket for traffic auditing and troubleshooting.
- Enable **CloudWatch metrics** for the target group to monitor `HealthyHostCount`, `RequestCount`, and `TargetResponseTime`.

**Scaling:**
- Register the EC2 instance into an **Auto Scaling Group** and attach the group to the target group. This enables the ALB to dynamically route to new instances as they are added and drain connections from instances being terminated.

**Security Hardening:**
- Add an **AWS WAF** web ACL to the ALB for protection against common web exploits (OWASP Top 10).
- Consider restricting the ALB security group to known CIDR ranges if public exposure is not required (e.g., internal-facing ALBs for internal services).































# AWS Application Load Balancer Web Ingress (EC2 + Nginx)

## Project Overview

- This project documents the deployment of an **internet-facing AWS Application Load Balancer (ALB)** that routes HTTP traffic to an existing **EC2 instance running Nginx** in the **us-east-1** region.

- The objective was to validate **end-to-end Layer 7 traffic flow** from the public internet through the ALB to the backend EC2 instance using **target groups and security group isolation**.

- This setup represents a foundational **web ingress architecture** commonly used in scalable cloud environments.

---

## Architecture Summary

             ┌────────────────────────┐
             │  Application Load      │
             │  Balancer (ALB)        │
             │  HTTP :80              │
             └───────────┬────────────┘
                         │
               ┌─────────▼─────────┐
               │ Target Group      │
               │ HTTP :80          │
               └─────────┬─────────┘
                         │
                ┌────────▼────────┐
                │ EC2 Instance    │
                │ devops-ec2      │
                │ Nginx :80       │
                └─────────────────┘


---

## 🔧 Technologies Used

* Amazon EC2
* Application Load Balancer (ALB)
* Target Groups
* Security Groups
* AWS CLI
* Nginx (Linux/UNIX)

---

## 🌍 Environment Details

| Component | Value |
|--------|------|
| Region | us-east-1 |
| VPC | Default VPC |
| EC2 Instance Name | devops-ec2 |
| Instance Type | t2.micro |
| Load Balancer | devops-alb |
| Target Group | devops-tg |
| ALB Security Group | devops-sg |

---

## 🚀 Implementation Steps

### Step 1: Discover Existing EC2 Instance

Identify the backend EC2 instance that will receive traffic.

`aws ec2 describe-instances`

```bash
Instance ID: i-011ae72a16b4398f6

VPC ID: vpc-0c3f56dfe99c83d0d

Subnet: subnet-00b5049732d6fd4eb

Security Group: default
```

📸 Screenshots: `EC2 instance details (running state)`
<img width="1039" height="805" alt="image" src="https://github.com/user-attachments/assets/9db485cd-3b2d-4b09-b3b9-d049498a51fb" />
<img width="1034" height="828" alt="image" src="https://github.com/user-attachments/assets/a96ff6d9-c1e7-4ef8-80cf-5222400f45b8" />
<img width="1033" height="829" alt="image" src="https://github.com/user-attachments/assets/e9412cb5-c775-4336-9de0-ca9f7f66fcde" />
<img width="1035" height="870" alt="image" src="https://github.com/user-attachments/assets/fb14a97a-917f-4b14-bd57-19e73b2ff7dc" />

###  Step 2: Confirm Default VPC
```bash
aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query "Vpcs[0].VpcId" \
  --output text
```

📸 Screenshot: `Default VPC verification` 
<img width="1041" height="368" alt="image" src="https://github.com/user-attachments/assets/8935f55e-8710-4c3d-9334-7b7f34e8b579" />

### Step 3: Create Security Group for ALB

Create a dedicated security group to allow public HTTP traffic.

```bash
aws ec2 create-security-group \
  --group-name devops-sg \
  --description "Security group for devops-alb" \
  --vpc-id vpc-0c3f56dfe99c83d0d

Allow inbound HTTP access:

aws ec2 authorize-security-group-ingress \
  --group-name devops-sg \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```

📸 Screenshot: `ALB security group creation` `Inbound rule allowing HTTP :80` 
<img width="1034" height="695" alt="image" src="https://github.com/user-attachments/assets/2d437a9b-50f3-4b8a-8f7f-8ce02c2c561e" />

### Step 4: Create Target Group

Define where the ALB will forward incoming traffic.

```bash
aws elbv2 create-target-group \
  --name devops-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-0c3f56dfe99c83d0d \
  --target-type instance
```

📸 Screenshot: `Target group configuration`
<img width="1039" height="863" alt="image" src="https://github.com/user-attachments/assets/e72780dc-56fa-48f8-9a0a-7308b2d2faee" />

### Step 5: Register EC2 Instance with Target Group
```bash
aws elbv2 register-targets \
  --target-group-arn <TARGET_GROUP_ARN> \
  --targets Id=i-011ae72a16b4398f6
```

📸 Screenshot: `Target instance registered and healthy`
<img width="1036" height="727" alt="image" src="https://github.com/user-attachments/assets/4782d0f9-177e-4c39-9f97-eb559ea407d2" />

### Step 6: Retrieve Subnets for ALB
```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0c3f56dfe99c83d0d" \
  --query "Subnets[*].SubnetId" \
  --output text
```

- Two subnets were selected to ensure high availability.

📸 Screenshot: `Subnets in default VPC`
<img width="1228" height="862" alt="image" src="https://github.com/user-attachments/assets/6802a510-d17f-4f87-ad93-eb2c49e6a645" />

### Step 7: Create Application Load Balancer
```bash
aws elbv2 create-load-balancer \
  --name devops-alb \
  --subnets subnet-09d0365261e4c44c6 subnet-00b5049732d6fd4eb \
  --security-groups sg-0f2b9556d703a6a3c
```

📸 Screenshot: `ALB successfully created`
<img width="1223" height="854" alt="image" src="https://github.com/user-attachments/assets/0589ef65-e44c-4742-8ec8-a196f8be51a1" />

### Step 8: Create HTTP Listener

- Configure ALB to forward HTTP traffic to the target group.

```bash
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN>
```

📸 Screenshot: `Listener forwarding HTTP :80 traffic`
<img width="1221" height="865" alt="image" src="https://github.com/user-attachments/assets/0212a8f0-806b-4eba-80d1-8834d412a0f8" />


### Step 9: Allow ALB Traffic to EC2

- Update EC2 security group to accept traffic only from the ALB.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id sg-0d5e8ddf23e6d5ab3 \
  --protocol tcp \
  --port 80 \
  --source-group sg-0f2b9556d703a6a3c
```

📸 Screenshot: `EC2 security group allowing ALB source`
<img width="1229" height="861" alt="image" src="https://github.com/user-attachments/assets/605fc323-a727-4452-bed5-c23254f06eb4" />

### 🔍 Validation Check
```bash
Retrieve ALB DNS Name
aws elbv2 describe-load-balancers \
  --names devops-alb \
  --query 'LoadBalancers[0].DNSName' \
  --output text
```

Expected Result:

`http://devops-alb-1274082417.us-east-1.elb.amazonaws.com`

📸 Screenshot: `Retrieved Application Load Balancer DNS Endpoint`
<img width="1215" height="862" alt="image" src="https://github.com/user-attachments/assets/38b8cdc3-dbf3-45b5-98c0-a909ca30eb99" />

### ✅ Validation Checklist

| Check                             | Status |
| --------------------------------- | ------ |
| EC2 instance running              | ✅      |
| ALB created                       | ✅      |
| Target group configured           | ✅      |
| EC2 registered as target          | ✅      |
| Listener forwarding traffic       | ✅      |
| Security group isolation enforced | ✅      |
| Application reachable via ALB     | ✅      |


### 🎯 Final Outcome

- Successfully implemented Layer 7 web ingress

- Public traffic routed securely through ALB

- Backend EC2 protected from direct internet exposure

- Architecture ready for Auto Scaling and HTTPS

- Matches production-grade AWS design patterns

### 🧠 Key Learnings

- ALB operates at Layer 7 (HTTP)

- Target groups decouple ingress from compute

- Security group chaining improves isolation

- ALB DNS endpoint becomes the public entry point

- Multi-subnet ALB enables high availability
