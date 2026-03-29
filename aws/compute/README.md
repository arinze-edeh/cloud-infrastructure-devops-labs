# AWS Compute Engineering Labs

![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![EC2](https://img.shields.io/badge/EC2-Compute-blue?logo=amazonaws)
![ALB](https://img.shields.io/badge/ALB-Load%20Balancing-green?logo=amazonaws)
![CLI](https://img.shields.io/badge/AWS%20CLI-Automation-yellow)
![Status](https://img.shields.io/badge/Status-Production--Ready-brightgreen)

---

## Overview

This directory contains hands-on AWS compute engineering projects executed entirely via the AWS CLI and Management Console. Each lab addresses a real-world infrastructure challenge: provisioning compute, hardening access, managing storage, scaling workloads, and maintaining operational discipline.

The work reflects the operational patterns expected in production cloud environments, including dynamic resource discovery, structured validation steps, CLI-driven automation, and documented troubleshooting.

---

## Directory Structure

```
compute/
├── ec2-cleanup/                        # EC2 instance termination and cleanup
├── ec2-ami-create/                     # AMI creation from existing EC2 instance
├── ec2-cli-launch-basics/              # Foundational EC2 provisioning via CLI
├── ec2-elastic-ip-setup/               # Elastic IP allocation and association
├── ec2-instance-resize/                # EC2 right-sizing for cost optimization
├── ec2-launch-secure/                  # Key pair creation and secure access setup
├── ec2-ssh-access/                     # SSH provisioning with user-data bootstrapping
├── ec2-stop-protection/                # Enabling stop protection via Console
├── ec2-storage-expansion/              # EBS volume provisioning via CLI
├── ec2-termination-protection/         # Enabling termination protection via Console
├── ebs-volume-attachment/              # Attaching EBS volumes to EC2 instances
├── high-availability-alb-ec2-deployment/ # HA web app with ASG and ALB
└── Resilient-Web-Tier-Architecture/    # ALB + EC2 Nginx deployment (Ubuntu)
```

---

## Project Summaries

---

### [High-Availability Web App: ASG + ALB](./high-availability-alb-ec2-deployment/)

**Quick Summary:** End-to-end implementation of a fault-tolerant, auto-scaling web application using EC2 Auto Scaling Groups, an internet-facing Application Load Balancer, and Nginx bootstrapped via User Data. Includes CPU-based target tracking and ELB-integrated health checks.

**Purpose:** Deliver a self-healing, scalable web tier that maintains minimum capacity, distributes traffic across healthy instances, and scales automatically under load.

**Approach:** Provisioned infrastructure in dependency order across 11 phases: security groups, AMI discovery, launch template with base64-encoded User Data, multi-AZ subnet selection, target group, ALB, listener, ASG, and TargetTracking scaling policy at 50% CPU. Used `amazon-linux-extras install nginx1 -y` to resolve the AL2 package repository gap. Set `health-check-grace-period 120` to absorb bootstrap time and `health-check-type ELB` to catch application-layer failures.

**Outcome:** Nginx serving traffic through the ALB DNS with confirmed `healthy` target state, active CloudWatch alarms (AlarmHigh and AlarmLow), and validated end-to-end HTTP 200 via `curl`.

**Key Decisions:**
- `$Latest` version reference in ASG ensures template changes propagate without ASG updates
- ELB health check type replaces default EC2 type for application-aware failure detection
- Two explicit AZ subnets selected rather than relying on auto-placement

---

### [Resilient Web Tier: ALB + EC2 Nginx (Ubuntu)](./resilient-web-tier-architecture/)

**Quick Summary:** Load-balanced Nginx deployment on Ubuntu 22.04, provisioned with a strict pre-flight discovery pattern. Documents a real AZ mismatch failure caused by EC2 auto-placement and its resolution.

**Purpose:** Deploy an internet-accessible Nginx web server behind an ALB using a security model that restricts direct EC2 access to ALB-originated traffic only.

**Approach:** Pre-collected all resource IDs before creating any infrastructure. Applied a split security group model: ALB uses the default SG with port 80 open to the internet; EC2 uses a dedicated SG that allows port 80 only from the ALB SG. Ubuntu user data validated for Unix line endings using `cat -A` before launch.

**Outcome:** HTTP 200 from ALB DNS with `Server: nginx/1.18.0 (Ubuntu)`. Includes a full post-mortem on the AZ mismatch issue, IAM permission gap discovery, and the remediation path (terminate, relaunch with explicit `--subnet-id`, re-register).

**Key Decisions:**
- Explicit `--subnet-id` at launch time to guarantee AZ alignment with the ALB
- Source-group-based ingress rule instead of CIDR for EC2 SG, eliminating direct internet exposure
- Pre-flight reference card approach prevents mid-deployment dependency failures

---

### [EC2 CLI Launch Basics](./ec2-cli-launch-basics/)

**Quick Summary:** Foundational EC2 provisioning workflow using the AWS CLI. Covers key pair creation, dynamic AMI resolution, VPC and subnet discovery, and instance lifecycle validation.

**Purpose:** Demonstrate repeatable, automation-ready EC2 provisioning without reliance on the AWS Console.

**Approach:** Dynamically resolved the latest Amazon Linux 2 AMI using `describe-images` with `sort_by` on `CreationDate`. Created and secured a key pair (`chmod 400`), identified the default VPC and subnet, launched a tagged instance, and verified running state via `describe-instances`.

**Outcome:** EC2 instance running in `us-east-1` with correct tagging, secured key pair, and state validated via CLI.

---

### [EC2 Elastic IP Setup](./ec2-elastic-ip-setup/)

**Quick Summary:** Allocated a VPC-scoped Elastic IP, associated it with a running EC2 instance, and validated that the public IP is persistent and correctly attached.

**Purpose:** Replace the ephemeral public IP assigned at launch with a static endpoint suitable for DNS and production use.

**Approach:** Verified identity with `aws sts get-caller-identity`, launched EC2 with inline tagging, allocated an EIP with a resource tag, associated it by `AllocationId`, and confirmed via both `describe-addresses` and `describe-instances` table output.

**Outcome:** Instance `i-0f6a39c883a6e3cb1` confirmed running with persistent Elastic IP attached and validated.

---

### [Secure EC2 SSH Access](./ec2-ssh-access/)

**Quick Summary:** Provisioned an EC2 instance with SSH key injection and root login enabled via a User Data bootstrap script. End-to-end validated with a successful `ssh root@<IP>` login.

**Purpose:** Automate secure SSH access configuration at instance launch, removing manual post-boot steps.

**Approach:** Generated an RSA key pair locally, embedded the public key into a User Data script that writes to `authorized_keys`, sets permissions, and configures `sshd_config` for root login. Security group restricted to TCP port 22. Validated with live SSH login.

**Outcome:** Root SSH access functional on first boot. Security notes document why root login is disabled in production and the preferred SSM Session Manager alternative.

---

### [EC2 Key Pair Creation](./ec2-launch-secure/)

**Quick Summary:** Created and secured an RSA EC2 key pair using the AWS CLI, applied `chmod 400`, and verified AWS-side registration.

**Purpose:** Establish a reusable, securely stored credential for EC2 SSH access as a standalone provisioning step.

**Approach:** Used `--query 'KeyMaterial'` with `--output text` to pipe the private key directly to a `.pem` file. Applied and verified `400` permissions with `ls -l`. Confirmed AWS registration with `describe-key-pairs`.

**Outcome:** Key pair registered in `us-east-1`, private key secured, ready for EC2 launch attachment.

---

### [EBS Volume Attachment](./ebs-volume-attachment/)

**Quick Summary:** Identified an existing EC2 instance and EBS volume by tag, attached the volume at `/dev/sdb`, and confirmed `attached` state via CLI.

**Purpose:** Extend block storage for a running EC2 instance as part of an incremental infrastructure migration.

**Approach:** Used tag-based filtering to resolve `InstanceId` and `VolumeId` into shell variables, then executed `attach-volume` with explicit device naming. Verified attachment state with `describe-volumes`.

**Outcome:** Volume attached at `/dev/sdb`, state confirmed as `attached` without instance restart.

---

### [EC2 Storage Expansion: EBS Volume Creation](./ec2-storage-expansion/)

**Quick Summary:** Provisioned a tagged 2 GiB gp3 EBS volume in `us-east-1` via CLI and validated size, type, and availability state.

**Purpose:** Provision block storage independently of compute to support incremental scaling with minimal disruption.

**Approach:** Listed available AZs, created the volume with `gp3` type and a `Name` tag, then verified all properties using `describe-volumes` before any attachment.

**Outcome:** `nautilus-volume` in `available` state, confirmed at 2 GiB gp3 with correct tagging.

---

### [EC2 AMI Creation](./ec2-ami-create/)

**Quick Summary:** Created an AMI named `nautilus-ec2-ami` from a running EC2 instance using the AWS Console and monitored status to `available`.

**Purpose:** Capture a point-in-time image of a production-configured instance to enable repeatable deployments, DR recovery, and scaling.

**Approach:** Selected the instance in EC2 console, triggered AMI creation via Actions, monitored AMI status until `available`, and validated ownership and readiness.

**Outcome:** AMI available and owned by the current account, ready for use in launch templates or direct instance launches.

---

### [EC2 Instance Right-Sizing](./ec2-instance-resize/)

**Quick Summary:** Safely downgraded an EC2 instance from `t2.micro` to `t2.nano` by stopping the instance, changing the type, and restarting with health check validation.

**Purpose:** Reduce compute costs for an underutilized instance without service disruption.

**Approach:** Verified instance health (2/2 status checks), stopped the instance, applied the type change via Instance Settings, restarted, and confirmed running state with the new type in place.

**Outcome:** Instance running as `t2.nano` with no downtime beyond the planned stop window. Reflects core FinOps practice: audit, right-size, validate.

---

### [EC2 Stop Protection](./ec2-stop-protection/)

**Quick Summary:** Enabled stop protection on a running EC2 instance via the AWS Console to prevent accidental state changes.

**Purpose:** Add a safety control to a critical instance to block unintended stop actions from the Console, CLI, or API.

**Approach:** Located `datacenter-ec2` in the EC2 Console, navigated to Instance Settings, enabled stop protection, and verified via the instance Details tab.

**Outcome:** Stop protection confirmed enabled. Accidental stops blocked across all access methods.

---

### [EC2 Termination Protection](./ec2-termination-protection/)

**Quick Summary:** Enabled termination protection on `devops-ec2` via the AWS Console to prevent accidental deletion.

**Purpose:** Protect a named production instance from permanent removal through an API or console-level safety control.

**Approach:** Navigated to EC2 Instances, selected the target, enabled termination protection via Instance Settings, and validated the configuration in the Details tab.

**Outcome:** Termination protection active. Instance cannot be deleted without explicitly disabling the protection flag first.

---

### [EC2 Cleanup](./ec2-cleanup/)

**Quick Summary:** Terminated a deprecated EC2 instance (`xfusion-ec2`) via the AWS Console and confirmed `terminated` state.

**Purpose:** Decommission an obsolete instance to eliminate unnecessary compute spend during infrastructure migration.

**Approach:** Located the instance by name in `us-east-1`, triggered termination, and waited for state confirmation.

**Outcome:** Instance reached `terminated` state. No residual compute costs from the decommissioned resource.

---

## Technologies and Tools

| Category | Tools |
|---|---|
| Cloud Platform | AWS (us-east-1) |
| Compute | EC2 (t2.micro, t2.nano), Auto Scaling Groups, Launch Templates |
| Networking | VPC, Subnets, Security Groups, Elastic IPs |
| Load Balancing | Application Load Balancer, Target Groups, Listeners |
| Storage | Amazon EBS (gp3), AMIs, Snapshots |
| Web Server | Nginx (Amazon Linux 2 via `amazon-linux-extras`, Ubuntu via `apt`) |
| Monitoring | CloudWatch (TargetTracking alarms: AlarmHigh, AlarmLow) |
| CLI Tooling | AWS CLI v2, `jq`-style query syntax, `base64`, `curl` |
| Access Management | EC2 Key Pairs, SSH, IAM (user-based) |
| OS | Amazon Linux 2, Ubuntu 22.04 LTS |

---

## Key Outcomes

**Infrastructure automation:** All provisioning steps executed via AWS CLI with dynamic resource discovery. No hardcoded IDs or manual Console lookups in the critical path.

**Resilience engineering:** Multi-AZ ALB and ASG configuration with ELB-integrated health checks, grace period tuning, and CPU-based TargetTracking scaling. Minimum one instance guaranteed at all times.

**Security posture:** Split security group model (ALB SG with public ingress, EC2 SG with source-group restriction), key pair secured at `400` permissions, stop and termination protection applied to sensitive instances.

**Operational discipline:** Pre-flight resource discovery before provisioning, structured validation at each phase, and documented troubleshooting for real failures encountered (AZ mismatch, IAM permission gaps, AL2 package repository issues).

**Cost management:** Demonstrated right-sizing workflow (t2.micro to t2.nano) and resource cleanup patterns that reflect production FinOps practices.

---

## How to Navigate

Each subdirectory contains a self-contained README with:
- Architecture overview or workflow logic
- Step-by-step CLI commands with expected outputs
- Screenshots at each validation checkpoint
- Lessons learned and troubleshooting reference

Start with **[high-availability-alb-ec2-deployment](./high-availability-alb-ec2-deployment/)** for the most comprehensive example of multi-component AWS infrastructure provisioning. Use **[Resilient-Web-Tier-Architecture](./Resilient-Web-Tier-Architecture/)** for a detailed breakdown of ALB + EC2 failure modes and real incident resolution.

The remaining projects are modular and can be read independently based on the specific skill area of interest.

---

> All labs were executed in `us-east-1` using IAM-scoped credentials. Resource IDs, account numbers, and ARNs shown in individual project READMEs reflect live lab environments and are not reusable across accounts.
