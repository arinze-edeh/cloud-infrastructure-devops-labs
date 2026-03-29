# AWS Networking

![AWS](https://img.shields.io/badge/AWS-Networking-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Region](https://img.shields.io/badge/Region-us--east--1-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![IaC](https://img.shields.io/badge/IaC-AWS%20CLI-0078D4?style=for-the-badge&logo=terminal&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

---

## Overview

This directory covers the core AWS networking domain, implemented entirely via the AWS CLI against live environments. Projects span VPC design, internet ingress, private subnet egress, cross-VPC connectivity, and access control, matching patterns used in multi-tier production architectures. Each task reflects a real operational scenario: onboarding a new VPC baseline, securing backend EC2 instances behind a load balancer, enabling private workloads to reach the internet without public exposure, and connecting isolated environments without traversing the public internet.

---

## Directory Structure

```
networking/
├── application-load-balancer-ec2-nginx/
├── aws-public-infrastructure-baseline/
├── ec2-eni-attachment/
├── inter-vpc-connectivity/
├── nat-gateway-private-subnet-egress/
├── private-subnet-egress-via-nat-instance/
├── security-groups-nacl/
└── subnet-creation-default-vpc/
```

---

## Project Summaries

### [Application Load Balancer + EC2 + Nginx](./application-load-balancer-ec2-nginx/)

**Quick Summary:** Deployed an internet-facing ALB routing HTTP traffic to a backend Nginx EC2 instance with full security group isolation between the load balancer and compute layers.

| | |
|---|---|
| **Purpose** | Validate end-to-end Layer 7 ingress from the public internet to a private EC2 backend |
| **Approach** | Created a dedicated ALB security group, target group, HTTP listener, and chained security group rules so the EC2 instance accepts traffic only from the ALB SG, not directly from the internet |
| **Outcome** | Public traffic routed through `devops-alb` DNS endpoint; EC2 shielded from direct internet exposure; architecture ready for Auto Scaling and HTTPS termination |

**Key decisions:** Security group chaining (SG-to-SG rather than CIDR-to-SG) enforces least-privilege isolation. Multi-subnet ALB placement enables high availability at no additional complexity cost.

---

### [AWS Public Infrastructure Baseline](./aws-public-infrastructure-baseline/)

**Quick Summary:** Built a reusable, non-default public VPC from scratch, including Internet Gateway, route table, public subnet with auto-assigned IPs, and a validated EC2 instance.

| | |
|---|---|
| **Purpose** | Establish a clean network foundation for future workloads, replacing reliance on the default VPC |
| **Approach** | Provisioned all components in strict dependency order: VPC, IGW, route table, subnet, security group, EC2. Explicit routing via `0.0.0.0/0 -> IGW` confirmed before compute launch |
| **Outcome** | Publicly accessible EC2 instance with SSH reachable; baseline marked ready for extension into multi-tier or private subnet architectures |

**Key decisions:** DNS hostnames and DNS support enabled at VPC creation to support future service discovery. Compute provisioned last to validate the network layer first.

---

### [EC2 ENI Attachment](./ec2-eni-attachment/)

**Quick Summary:** Attached an existing Elastic Network Interface to a running EC2 instance using the AWS CLI and confirmed `in-use` status via describe output.

| | |
|---|---|
| **Purpose** | Support an incremental AWS migration requiring an ENI to be hot-attached to a live instance |
| **Approach** | Retrieved instance ID and ENI ID via tag filters, attached at device index 1, verified attachment status without instance restart |
| **Outcome** | ENI attached and confirmed `in-use`; instance continued serving traffic without interruption |

**Key decisions:** Tag-based resource lookup over hardcoded IDs keeps commands repeatable across lab resets.

---

### [Inter-VPC Connectivity (VPC Peering)](./inter-vpc-connectivity/)

**Quick Summary:** Configured VPC Peering between the default VPC (`172.31.0.0/16`) and a private VPC (`10.1.0.0/16`) to achieve private ICMP connectivity with 0% packet loss, resolving four sequential blockers across peering, routing, security groups, and SSH key trust.

| | |
|---|---|
| **Purpose** | Enable private communication between isolated VPC environments without internet traversal, a common pattern for dev/staging/prod separation and multi-tier microservices |
| **Approach** | Created and accepted the peering connection, added bidirectional routes in both VPCs' main route tables, opened ICMP on the private EC2 SG, and bootstrapped SSH key trust via EC2 Instance Connect to execute the final ping test |
| **Outcome** | `ping nautilus-private-ec2` from `nautilus-public-ec2`: 4/4 packets received, 0% packet loss, avg RTT 1.53ms |

**Key decisions:** All resource IDs resolved into shell variables upfront to eliminate hardcoding and support reuse. EC2 Instance Connect used as a keyless bootstrap mechanism when no key pair existed on the instance. `$(cat ...)` local expansion used instead of piped `cat` to avoid remote stdin blocking.

---

### [NAT Gateway - Private Subnet Egress](./nat-gateway-private-subnet-egress/)

**Quick Summary:** Deployed a managed NAT Gateway to provide controlled outbound internet access from a private subnet, verified by a cron-driven S3 upload from the private EC2 instance.

| | |
|---|---|
| **Purpose** | Allow a private EC2 workload to reach the internet for artifact uploads without assigning a public IP |
| **Approach** | Created a public subnet in the same AZ as the private subnet, provisioned an IGW, configured a public route table, allocated an Elastic IP, deployed the NAT Gateway, and updated the private route table to direct `0.0.0.0/0` through the NAT Gateway |
| **Outcome** | `xfusion-test.txt` confirmed present in `s3://xfusion-nat-3445/` within minutes of routing completion |

**Key decisions:** NAT Gateway placed in the same AZ as the private subnet to eliminate cross-AZ data transfer costs. `aws ec2 wait nat-gateway-available` used to prevent race conditions on route table creation. S3 upload treated as the true end-to-end validation, not just infrastructure state checks.

---

### [Private Subnet Egress via NAT Instance](./private-subnet-egress-via-nat-instance/)

**Quick Summary:** Deployed a self-managed NAT Instance (Amazon Linux 2 with iptables MASQUERADE) as a cost-optimized alternative to the managed NAT Gateway, verified by the same cron-driven S3 upload pattern.

| | |
|---|---|
| **Purpose** | Enable outbound internet access from a private subnet when NAT Gateway costs are not justified for the workload |
| **Approach** | Launched an Amazon Linux 2 EC2 in a new public subnet, disabled Source/Destination Check, configured `net.ipv4.ip_forward=1` and iptables MASQUERADE on `eth0`, persisted rules via `iptables-save`, and updated the private route table to target the NAT instance ENI |
| **Outcome** | `devops-test.txt` confirmed in `s3://devops-nat-17499/`; NAT Instance processing traffic from private EC2 at `10.1.1.x` |

**Key decisions:** `iptables-save | tee /etc/sysconfig/iptables` used instead of `service iptables save` (unavailable on Amazon Linux 2 systemd). EC2 Instance Connect used to bootstrap SSH access without a pre-existing key pair. NAT Instance documented as a cost trade-off: lower hourly cost, higher operational overhead and single point of failure versus managed NAT Gateway.

---

### [Security Groups and NACLs](./security-groups-nacl/)

**Quick Summary:** Created and configured a VPC security group with explicit inbound rules for HTTP (80) and SSH (22), validated via CLI describe output.

| | |
|---|---|
| **Purpose** | Establish network-level access control for EC2 instances as a prerequisite for compute deployment |
| **Approach** | Queried the default VPC, created a scoped security group, added least-privilege inbound rules, and verified configuration via `describe-security-groups` |
| **Outcome** | Security group ready for EC2 attachment with only required ports exposed |

**Key decisions:** SSH rule scoped to `0.0.0.0/0` for lab accessibility; documented with a production note to restrict to a known bastion IP or remove in favour of EC2 Instance Connect.

---

### [Subnet Creation in Default VPC](./subnet-creation-default-vpc/)

**Quick Summary:** Provisioned a new subnet inside the default VPC with CIDR conflict validation, AZ placement, and resource tagging.

| | |
|---|---|
| **Purpose** | Add network segmentation to the default VPC before deploying compute or RDS workloads |
| **Approach** | Retrieved existing subnet CIDRs to avoid conflicts, created a `/20` subnet in `us-east-1a`, applied a `Name` tag, and verified via describe output |
| **Outcome** | Subnet `devops-subnet` confirmed available in the correct AZ with the correct CIDR |

**Key decisions:** CIDR audit performed before creation to prevent silent overlap failures. Tagging applied immediately post-creation for operational traceability.

---

## Technologies and Tools

| Category | Tools / Services |
|---|---|
| Networking | VPC, Subnets, Internet Gateway, NAT Gateway, NAT Instance, Route Tables, VPC Peering, Elastic IP |
| Load Balancing | Application Load Balancer, Target Groups, Listeners |
| Compute | EC2 (Amazon Linux 2, various instance types) |
| Access Control | Security Groups, NACLs, EC2 Instance Connect |
| Network Interfaces | Elastic Network Interface (ENI) |
| Storage | S3 (used for egress validation) |
| OS Networking | iptables, `sysctl`, `ip_forward`, MASQUERADE |
| Tooling | AWS CLI v2, Bash, shell variable composition |

---

## Key Outcomes and Skills Demonstrated

- Designed and validated multi-layer network architectures from VPC baseline through ALB ingress and private subnet egress
- Debugged cross-layer connectivity failures spanning peering, routing, security groups, and SSH key trust, each resolved with CLI-only tooling
- Applied cost-aware architecture decisions: NAT Instance vs. NAT Gateway trade-off analysis documented with operational implications
- Enforced security group chaining and least-privilege ingress patterns consistent with production standards
- Used tag-based resource lookup, shell variable composition, and `aws ec2 wait` consistently across all projects for repeatable, race-condition-safe execution
- Validated all outcomes at the application layer (S3 upload, HTTP response, ICMP ping) rather than infrastructure state alone

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with full implementation steps, exact CLI commands, expected outputs, error resolution logs, and screenshot references. Projects can be read independently. For a logical progression through the domain:

1. `subnet-creation-default-vpc` and `aws-public-infrastructure-baseline` cover VPC fundamentals
2. `security-groups-nacl` and `ec2-eni-attachment` cover instance-level access control and network interfaces
3. `nat-gateway-private-subnet-egress` and `private-subnet-egress-via-nat-instance` cover private subnet egress patterns
4. `inter-vpc-connectivity` covers multi-VPC network design
5. `application-load-balancer-ec2-nginx` covers production ingress architecture

---

> All commands, resource IDs, and outputs across these projects reflect live implementation runs. No values have been approximated or substituted.
