# monitoring-and-logging

> AWS observability, log pipeline automation, and threshold-based alerting implemented via CLI-first workflows and IAM-native access patterns.

---

## Overview

Centralized log management and real-time alerting are baseline requirements in any production AWS environment. This directory covers two distinct but complementary patterns: automated cross-VPC log ingestion into S3 using VPC Peering and IAM instance profiles, and proactive CPU threshold alerting using CloudWatch and SNS. Both projects reflect the operational standards expected in environments where manual intervention is not an option and auditability is required.

---

## Directory Structure

```
monitoring-and-logging/
├── cross-vpc-log-ingestion-and-s3-archival-pipeline/
│   └── README.md
├── ec2-performance-threshold-alerting/
│   └── README.md
└── README.md
```

---

## Project Summaries

---

### cross-vpc-log-ingestion-and-s3-archival-pipeline

**Quick Summary**
Automated pipeline that ships boot logs from a private, internet-isolated EC2 instance across a VPC Peering connection to a public relay instance, which archives them to a private S3 bucket via IAM instance profile. Zero manual intervention after initial setup.

**Purpose**
Address the operational requirement of centralizing logs from isolated compute with no NAT Gateway, no internet exposure, and no static credentials.

**Approach**
- Built and peered two non-overlapping VPCs (`10.1.0.0/16`, `10.10.0.0/16`) with bidirectional route table entries
- Attached an IAM instance profile (`devops-s3-role`) to the public EC2, enabling credential-free S3 writes via the instance metadata endpoint
- Configured cron jobs on both instances: private EC2 SCPs `/var/log/boots.log` every minute over the peering link; public EC2 runs `aws s3 cp` on the same schedule to push to `s3://devops-s3-logs-25406/devops-priv-vpc/boot/`
- Enforced S3 private access with `put-public-access-block` across all four block settings

**Key Decisions**
- VPC Peering over NAT Gateway: lower cost, no internet traversal, appropriate for VPC-to-VPC log relay
- IAM instance profile over access keys: eliminates credential rotation risk on long-lived instances
- Absolute binary path (`/usr/local/bin/aws`) in cron: prevents silent failures from a minimal cron PATH environment

**Outcome**
End-to-end pipeline confirmed operational. `boots.log` (23 bytes) lands at the correct S3 path within 60 seconds of pipeline activation. VPC Peering status validated as `active`.

---

### ec2-performance-threshold-alerting

**Quick Summary**
CloudWatch alarm configured to trigger on CPU utilization reaching 90%, with SNS fan-out for notification delivery. EC2 health and alarm state validated via CLI.

**Purpose**
Establish a reproducible, CLI-driven monitoring baseline for EC2 workloads where reactive incident awareness is required without a third-party observability tool.

**Approach**
- Launched a tagged Ubuntu 22.04 `t2.micro` instance (`nautilus-ec2`) using the latest Canonical AMI
- Created a CloudWatch alarm (`nautilus-alarm`) with a 5-minute evaluation period, single datapoint threshold, and `GreaterThanOrEqualToThreshold` comparison against `CPUUtilization` in the `AWS/EC2` namespace
- Bound the alarm action to an existing SNS topic ARN retrieved via CLI query
- Validated alarm state (`OK`) and EC2 health checks (system + instance) post-deployment

**Key Decisions**
- Single evaluation period: suitable for burst-sensitive workloads where a sustained 5-minute breach is the alert condition
- CLI-first throughout: all steps are reproducible without console access, supporting IaC and automation pipelines

**Outcome**
Alarm active, EC2 running, SNS action attached. Full validation confirmed via `describe-alarms` and `describe-instance-status`.

---

## Technologies and Tools

| Category | Tools / Services |
|---|---|
| Compute | Amazon EC2 (Ubuntu 22.04 LTS, t2.micro) |
| Networking | VPC Peering, Internet Gateway, Security Groups, Route Tables |
| Storage | Amazon S3 (private bucket, path-based log archival) |
| IAM | Instance profiles, managed policies, trust relationships |
| Monitoring | Amazon CloudWatch (metrics, alarms, evaluation periods) |
| Notifications | Amazon SNS |
| Automation | cron, SCP, AWS CLI v2 |
| CLI | AWS CLI v2 (`aws ec2`, `aws s3`, `aws cloudwatch`, `aws iam`, `aws sns`) |

---

## Key Outcomes and Skills Demonstrated

- **Cross-VPC networking**: VPC Peering setup with bidirectional route table configuration and security group scoping to VPC CIDR ranges
- **IAM-native access patterns**: Instance profile attachment and credential-free S3 access via IMDS; no static keys used
- **Pipeline automation**: Dual-cron log relay with absolute path resolution and pre-populated `authorized_keys` for non-interactive SCP
- **S3 security posture**: Public access block enforcement at bucket level as default configuration
- **CloudWatch alerting**: Metric alarm configuration with SNS action binding and post-deployment state validation
- **CLI reproducibility**: All infrastructure provisioned and validated exclusively via AWS CLI; no console dependency

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with:
- Full CLI command sequences with expected outputs
- Inline screenshot placeholders at each step
- A resource reference table with all IDs and values used
- A troubleshooting section covering the most common failure modes
- Best practices and lessons learned sections

Start with `cross-vpc-log-ingestion-and-s3-archival-pipeline` for a complete end-to-end pipeline reference, or `ec2-performance-threshold-alerting` for a focused CloudWatch and SNS integration walkthrough.

---

> Part of the [cloud-infrastructure-devops-labs](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs) portfolio | AWS | `us-east-1`
