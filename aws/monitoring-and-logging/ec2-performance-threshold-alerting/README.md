# EC2 CPU Utilization Monitoring with CloudWatch Threshold Alerting

> **Production-grade observability setup:** Monitor EC2 compute resources, enforce CPU utilization thresholds, and deliver incident notifications through SNS, all driven through the AWS CLI.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Validate AWS Identity and Region](#step-1-validate-aws-identity-and-region)
  - [Step 2: Discover Latest Ubuntu 22.04 AMI](#step-2-discover-latest-ubuntu-2204-ami)
  - [Step 3: Launch EC2 Instance](#step-3-launch-ec2-instance)
  - [Step 4: Retrieve SNS Topic ARN](#step-4-retrieve-sns-topic-arn)
  - [Step 5: Create CloudWatch CPU Alarm](#step-5-create-cloudwatch-cpu-alarm)
  - [Step 6: Validate Alarm Configuration](#step-6-validate-alarm-configuration)
  - [Step 7: Confirm EC2 Health Status](#step-7-confirm-ec2-health-status)
- [Final Outcome](#final-outcome)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Design Notes](#design-notes)

---

## Project Overview

This project implements a **production-aligned monitoring pipeline** for an Amazon EC2 instance using **Amazon CloudWatch** and **SNS-based alerting**. The setup ensures that when CPU utilization breaches a defined operational threshold, on-call or operations teams receive timely notifications, enabling proactive incident response before end users are impacted.

All steps are executed via the **AWS CLI**, making this implementation fully reproducible, auditable, and pipeline-friendly. No console clicks. No assumptions.

---

## Problem Statement

Undetected compute saturation on EC2 instances is a leading cause of degraded application performance and unplanned outages. Without proactive threshold-based alerting:

- CPU spikes go unnoticed until user impact occurs
- Incident response is reactive rather than preventive
- Engineering teams lack visibility into workload health

**Solution:** Instrument EC2 instances with CloudWatch metric alarms tied to SNS topics, ensuring operational awareness the moment resource utilization crosses acceptable bounds.

---

## Architecture

```
EC2 Instance (Ubuntu 22.04, t2.micro)
  └── CloudWatch Metric: CPUUtilization (AWS/EC2 namespace)
        └── CloudWatch Alarm: nautilus-alarm (threshold >= 90%, 5-min period)
              └── SNS Topic: nautilus-sns-topic
                    └── Notification Fan-out (email, Lambda, PagerDuty, etc.)
```

**Key Design Decisions:**

- **Metric:** `CPUUtilization` from the `AWS/EC2` namespace, natively emitted without an agent
- **Statistic:** `Average` over a 5-minute period, reducing noise from momentary spikes
- **Evaluation window:** 1 consecutive datapoint breaching threshold to trigger alarm, balancing speed vs. false positives
- **Notification channel:** Pre-existing SNS topic decouples alarm logic from subscriber management

---

## Prerequisites

Before beginning, confirm the following are in place:

- **AWS CLI** installed and configured (`aws configure`)
- **IAM permissions** for:
  - `ec2:DescribeImages`, `ec2:RunInstances`, `ec2:DescribeInstanceStatus`
  - `cloudwatch:PutMetricAlarm`, `cloudwatch:DescribeAlarms`
  - `sns:ListTopics`
- **Existing SNS topic** named `nautilus-sns-topic` with at least one active subscription
- **Default region** set to `us-east-1`
- **STS access** for identity verification

---

## Implementation

---

### Step 1: Validate AWS Identity and Region

Before executing any provisioning commands, confirm the active IAM identity and account context. This prevents misrouted resource creation due to misconfigured profiles or credential conflicts.

```bash
aws sts get-caller-identity
```

**Expected Output:**

```json
{
  "UserId": "<IAM_USER_ID>",
  "Account": "<AWS_ACCOUNT_ID>",
  "Arn": "arn:aws:iam::<ACCOUNT_ID>:user/<USERNAME>"
}
```

**Validation Criteria:**

- The `Account` field matches the intended target AWS account
- The `Arn` reflects the correct IAM user or role
- The CLI region is `us-east-1` (verify with `aws configure get region`)

> **Best Practice:** In multi-account environments, always verify identity before provisioning. A misplaced EC2 instance in the wrong account can be costly and difficult to trace.

**Screenshot: AWS STS identity confirmation**

![Step 1 - AWS Identity Validation](https://github.com/user-attachments/assets/4e982204-50b9-4b62-9c55-fe33589e8ac5)

*The output confirms the active IAM user (`kk_labs_user_365845`), account ID (`375398625824`), and that the CLI is operating in `us-east-1`.*

---

### Step 2: Discover Latest Ubuntu 22.04 AMI

Rather than hardcoding an AMI ID (which becomes stale as Canonical publishes updates), dynamically retrieve the most recently published Ubuntu 22.04 LTS (Jammy) AMI from the official Canonical AWS account (`099720109477`).

```bash
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```

**Expected Output:**

```
ami-04680790a315cd58d
```

**Key Parameters:**

- `--owners 099720109477`: Scopes the search to Canonical's official AWS account, preventing use of unverified community AMIs
- `sort_by(...CreationDate)[-1]`: Selects the most recently published image, ensuring up-to-date OS patches
- `--output text`: Returns a clean, scriptable string for downstream substitution

> **Security Note:** Always pin AMI discovery to the canonical owner ID. Using community AMIs without verification is a supply chain risk.

**Screenshot: Latest Ubuntu 22.04 AMI ID retrieved dynamically**

![Step 2 - AMI Discovery](https://github.com/user-attachments/assets/18aaebac-3a33-4963-bdbe-2c9039e5763e)

*The query returns AMI `ami-04680790a315cd58d`, the most recently published Ubuntu 22.04 LTS image available in `us-east-1`.*

---

### Step 3: Launch EC2 Instance

Launch a `t2.micro` instance using the retrieved AMI, applying a descriptive name tag for resource identification and cost attribution.

```bash
aws ec2 run-instances \
  --image-id ami-04680790a315cd58d \
  --count 1 \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]'
```

**Expected Outcome:**

- Instance enters `pending` state immediately upon launch
- A `ReservationId` and `InstanceId` (`i-0deff0fe61feb1d27`) are returned in the JSON response
- The instance is tagged `Name=nautilus-ec2` for identification in the console and CLI queries
- Instance type is `t2.micro` (eligible for AWS Free Tier)

**Notable Response Fields to Record:**

| Field | Value |
|---|---|
| `InstanceId` | `i-0deff0fe61feb1d27` |
| `ImageId` | `ami-04680790a315cd58d` |
| `InstanceType` | `t2.micro` |
| `AvailabilityZone` | `us-east-1c` |
| `PrivateIpAddress` | `172.31.28.164` |
| `SubnetId` | `subnet-0ca1517af522634e6` |

> **Operational Note:** Store the `InstanceId` returned in the response. It is required for alarm dimension configuration in Step 5 and health validation in Step 7.

**Screenshot: EC2 run-instances response (initial JSON output)**

![Step 3a - Instance Launch Response](https://github.com/user-attachments/assets/0e59112a-a29a-4ec1-8487-1afe5d85d9a5)

*The `run-instances` command returns a detailed JSON payload confirming instance creation. The `ReservationId`, `InstanceId`, network interface details, and security group assignments are visible.*

**Screenshot: EC2 network interface and subnet details**

![Step 3b - Network Interface Details](https://github.com/user-attachments/assets/521d1d1c-97c2-46e0-9c6a-87db195b9fb9)

*The response confirms the instance's private IP (`172.31.28.164`), subnet (`subnet-0ca1517af522634e6`), VPC (`vpc-07995b2baf90b84da`), and tag (`Name: nautilus-ec2`).*

**Screenshot: Instance metadata, InstanceId, and pending state**

![Step 3c - Instance ID and State](https://github.com/user-attachments/assets/ff03585c-72da-48e9-8ce7-d96de01b2279)

*`InstanceId: i-0deff0fe61feb1d27` is confirmed. The instance state shows `pending` immediately post-launch, which is expected. The `LaunchTime` is recorded as `2026-02-26T01:07:55.000Z`.*

**Screenshot: Instance capacity and placement details**

![Step 3d - Placement and Capacity](https://github.com/user-attachments/assets/13be1bcd-d154-4e93-b1da-f1d8ef4155a0)

*Final section of the `run-instances` response shows placement in `us-east-1c`, `InstanceType: t2.micro`, and boot mode as `legacy-bios`. Monitoring is initially `disabled` (CloudWatch detailed monitoring is not enabled by default for `t2.micro`).*

---

### Step 4: Retrieve SNS Topic ARN

Retrieve the ARN of the pre-existing SNS topic that will receive alarm notifications. Using a JMESPath filter ensures the correct topic is returned even in accounts with multiple SNS topics.

```bash
TOPIC_ARN=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn, 'nautilus-sns-topic')].TopicArn" \
  --output text)
echo "SNS Topic ARN: $TOPIC_ARN"
```

**Expected Output:**

```
SNS Topic ARN: arn:aws:sns:us-east-1:375398625824:nautilus-sns-topic
```

**Operational Notes:**

- Storing the ARN as `$TOPIC_ARN` enables clean reuse in the subsequent alarm creation command
- The `contains()` JMESPath filter is resilient to partial name matches; use exact matching in production (`== 'nautilus-sns-topic'`) if topic names are non-unique
- Verify the topic has active subscriptions before attaching it to an alarm; a topic with no confirmed subscribers will silently drop notifications

> **Best Practice:** Validate SNS subscriptions are confirmed (`aws sns list-subscriptions-by-topic --topic-arn $TOPIC_ARN`) before wiring the topic to alarms in production systems.

**Screenshot: SNS Topic ARN retrieved and stored**

![Step 4 - SNS Topic ARN](https://github.com/user-attachments/assets/0bdc588b-9ba4-47d6-b8db-d35684db0fb2)

*The command returns `arn:aws:sns:us-east-1:375398625824:nautilus-sns-topic`, confirming the topic exists in the correct account and region. The ARN is echoed for verification.*

---

### Step 5: Create CloudWatch CPU Alarm

Create a CloudWatch metric alarm scoped to the specific EC2 instance. The alarm fires when average CPU utilization equals or exceeds **90%** over a single 5-minute evaluation period.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "nautilus-alarm" \
  --alarm-description "Alarm when CPU exceeds 90 percent" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=InstanceId,Value=i-0deff0fe61feb1d27 \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:375398625824:nautilus-sns-topic \
  --unit Percent
```

**Alarm Parameter Reference:**

| Parameter | Value | Rationale |
|---|---|---|
| `--metric-name` | `CPUUtilization` | Native EC2 metric, no agent required |
| `--namespace` | `AWS/EC2` | Standard CloudWatch namespace for EC2 |
| `--statistic` | `Average` | Smooths transient spikes, reflects sustained load |
| `--period` | `300` | 5-minute evaluation window (minimum recommended for production) |
| `--threshold` | `90` | 90% CPU, indicating near-saturation |
| `--comparison-operator` | `GreaterThanOrEqualToThreshold` | Inclusive threshold breach detection |
| `--evaluation-periods` | `1` | Single datapoint trigger, maximizes alert speed |
| `--alarm-actions` | SNS Topic ARN | Routes notification to SNS fan-out |

> **Tuning Considerations:** In production, consider setting `--evaluation-periods` to `2` or `3` to reduce false positives from brief CPU bursts (e.g., OS updates, cron jobs). A single-period evaluation maximizes sensitivity but may generate alert noise on burstable instance types like `t2.micro`.

> **Edge Case:** `t2.micro` uses CPU credits. High utilization may be transient (credit exhaustion). Consider monitoring `CPUCreditBalance` alongside `CPUUtilization` for burstable instances.

**Screenshot: CloudWatch put-metric-alarm command execution**

![Step 5 - CloudWatch Alarm Creation](https://github.com/user-attachments/assets/6755463a-6c02-4807-8ff9-74eaf005a391)

*The complete `put-metric-alarm` command is issued with all parameters, including the resolved `InstanceId` (`i-0deff0fe61feb1d27`) and SNS Topic ARN. A successful execution returns no output, indicating the alarm was created without errors.*

---

### Step 6: Validate Alarm Configuration

Immediately after creation, describe the alarm to confirm its configuration, current state, and SNS action attachment. This step validates that the alarm is correctly wired before any traffic or load is applied.

```bash
aws cloudwatch describe-alarms \
  --alarm-names "nautilus-alarm"
```

**Expected Validation Criteria:**

- `AlarmName` matches `nautilus-alarm`
- `StateValue` is `OK` (no threshold breach detected)
- `AlarmActions` contains the correct SNS Topic ARN
- `MetricName` is `CPUUtilization`
- `Threshold` is `90.0`
- `ComparisonOperator` is `GreaterThanOrEqualToThreshold`
- `EvaluationPeriods` is `1`
- `ActionsEnabled` is `true`

**Confirmed Alarm Configuration from Output:**

```json
{
  "AlarmName": "nautilus-alarm",
  "AlarmDescription": "Alarm when CPU exceeds 90 percent",
  "StateValue": "OK",
  "AlarmActions": ["arn:aws:sns:us-east-1:375398625824:nautilus-sns-topic"],
  "MetricName": "CPUUtilization",
  "Namespace": "AWS/EC2",
  "Statistic": "Average",
  "Period": 300,
  "Threshold": 90.0,
  "ComparisonOperator": "GreaterThanOrEqualToThreshold",
  "EvaluationPeriods": 1
}
```

> **Note:** The `StateReason` in the output confirms the alarm evaluated `4.29%` average CPU, far below the `90%` threshold, placing it immediately in `OK` state. This is expected behavior for a newly launched, idle instance.

**Screenshot: CloudWatch alarm description confirming correct configuration and OK state**

![Step 6 - Alarm Validation](https://github.com/user-attachments/assets/1c10d3b0-f196-44c7-9a6a-df8c85d67d69)

*The `describe-alarms` output confirms `StateValue: OK`, `Threshold: 90.0`, `ActionsEnabled: true`, and the SNS topic ARN is correctly attached as the alarm action. The `StateTransitionedTimestamp` shows the alarm evaluated and transitioned to OK at `2026-02-26T01:13:07.083Z`.*

---

### Step 7: Confirm EC2 Health Status

After the instance has had time to initialize (typically 2 to 3 minutes post-launch), verify that both EC2 system-level and instance-level health checks have passed.

```bash
aws ec2 describe-instance-status \
  --instance-ids i-0deff0fe61feb1d27
```

**Expected Validation Criteria:**

- `InstanceState.Name` is `running`
- `InstanceStatus.Status` is `ok`
- `SystemStatus.Status` is `ok`
- Both `reachability` checks show `passed`

> **Timing Note:** Health check results are not immediately available. If the instance was just launched, wait 60 to 90 seconds and retry. A `running` instance does not guarantee health checks have completed.

> **Troubleshooting:** If either status check returns `impaired`, investigate via the EC2 console event log or `aws ec2 describe-instance-status --include-all-instances`. Common causes include kernel panics, failed disk mounts, or hypervisor hardware degradation (system check failures).

**Screenshot: EC2 instance health status showing both checks passed**

![Step 7 - EC2 Health Validation](https://github.com/user-attachments/assets/30c07378-729f-4293-a968-27ad66cd6d29)

*The output confirms `InstanceState: running` (Code 16), `InstanceStatus: ok`, and `SystemStatus: ok`. Both `reachability` checks show `passed`, confirming the instance is fully operational and healthy in `us-east-1c`.*

---

## Final Outcome

The following production objectives were achieved:

- **EC2 instance deployed:** `i-0deff0fe61feb1d27` (`nautilus-ec2`) running Ubuntu 22.04 in `us-east-1c`
- **CloudWatch alarm active:** `nautilus-alarm` monitoring `CPUUtilization` with a `>= 90%` threshold
- **SNS notification configured:** `arn:aws:sns:us-east-1:375398625824:nautilus-sns-topic` attached as alarm action
- **Alarm state validated:** `StateValue: OK`, confirming correct baseline behavior
- **Instance health confirmed:** Both system and instance status checks report `ok`

---

## Operational Considerations

**Alarm Sensitivity Tuning**

For production workloads, consider increasing `--evaluation-periods` to `2` or `3` to reduce alarm fatigue from short-lived spikes. For SLA-critical systems, reduce the period from `300` to `60` seconds for faster detection.

**Burstable Instance Types**

`t2.micro` and other T-series instances use a CPU credit model. Sustained CPU above the baseline (10% for `t2.micro`) depletes credits. Monitor `CPUCreditBalance` alongside `CPUUtilization` to detect impending throttling before it occurs.

**SNS Subscription Management**

Ensure SNS subscriptions are confirmed prior to relying on alarm notifications. Unconfirmed email subscriptions do not receive messages. Use `aws sns list-subscriptions-by-topic` to audit subscriber state.

**CloudWatch Basic vs. Detailed Monitoring**

By default, EC2 instances emit metrics at 5-minute intervals (basic monitoring). For 1-minute resolution, enable detailed monitoring: `aws ec2 monitor-instances --instance-ids <id>`. Note this incurs additional CloudWatch costs.

**IAM Least Privilege**

The IAM user or role running these commands should be scoped to the minimum required permissions. Avoid using broad `AdministratorAccess` policies in production environments.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `describe-alarms` returns empty `MetricAlarms` | Alarm name typo or wrong region | Confirm `--alarm-names` value and active region |
| Alarm stuck in `INSUFFICIENT_DATA` | Metrics not yet flowing for new instance | Wait 5 to 10 minutes for CloudWatch to receive initial datapoints |
| SNS notification not received | Subscription unconfirmed or wrong topic ARN | Check `aws sns list-subscriptions-by-topic` for confirmed subscriptions |
| Instance health checks `impaired` | Kernel panic or hardware degradation | Review EC2 system logs via console; consider stopping and restarting the instance |
| AMI query returns no results | Region has no matching image or wrong owner | Confirm `--owners 099720109477` and region; retry without the date wildcard |
| `put-metric-alarm` returns `InvalidFormat` | Malformed ARN or invalid dimension value | Verify SNS ARN format and that `InstanceId` exists in the region |

---

## Cleanup

Remove all provisioned resources after validation to avoid unnecessary charges:

```bash
# Delete the CloudWatch alarm
aws cloudwatch delete-alarms --alarm-names nautilus-alarm

# Terminate the EC2 instance
aws ec2 terminate-instances --instance-ids i-0deff0fe61feb1d27
```

**Verification after cleanup:**

```bash
# Confirm alarm is removed
aws cloudwatch describe-alarms --alarm-names "nautilus-alarm"

# Confirm instance is terminating or terminated
aws ec2 describe-instances \
  --instance-ids i-0deff0fe61feb1d27 \
  --query 'Reservations[].Instances[].State.Name'
```

---

## Design Notes

- **Monitoring-first architecture:** Observability is provisioned alongside the compute resource, not retroactively. This reflects production-grade engineering discipline.
- **CLI-driven reproducibility:** Every step is executable from a terminal, suitable for embedding in CI/CD pipelines, runbooks, or IaC bootstrapping scripts.
- **Threshold-based alerting:** Alarm thresholds are intentionally conservative (90% CPU) to signal near-saturation rather than sustained overload, enabling proactive response.
- **Minimal assumptions:** No VPC customizations, key pairs, or security group modifications are required. The implementation uses AWS account defaults, making it portable across environments.
- **Production-aligned observability pattern:** SNS fan-out decouples alarm logic from notification channel management, enabling email, Lambda, PagerDuty, or Slack integrations without modifying the alarm itself.































# EC2 Performance Threshold Alerting with CloudWatch

## Project Overview

This project demonstrates a **production-style monitoring setup** where an **EC2 instance** is monitored using **Amazon CloudWatch**, and an **alarm is triggered when CPU utilization exceeds a defined threshold**.  

Upon breach, notifications are sent via an existing **SNS topic**, enabling proactive incident awareness.

The implementation follows **CLI-first**, **least-assumption**, and **Operational-style documentation** principles.

---

## Scope & Objectives

* Provision an Ubuntu EC2 instance
* Monitor EC2 CPU utilization
* Trigger alerts when CPU usage ≥ **90%**
* Send notifications using an SNS topic
* Validate alarm state and instance health

---

## Architecture Overview


- EC2 (Ubuntu 22.04)
- └── CloudWatch Metric (CPUUtilization)
- └── CloudWatch Alarm (Threshold ≥ 90%)
- └── SNS Topic (Notification Fan-out)

---

## Prerequisites

* AWS CLI installed and authenticated
* IAM user with:
  * EC2 permissions
  * CloudWatch permissions
  * SNS permissions
* Existing SNS topic
* Region set to **us-east-1**

---

## Implementation Steps (CLI-First)

---

### Step 1️: Validate AWS Identity & Region

```
aws sts get-caller-identity
```
#### Expected Outcome

- Valid IAM identity returned

- Correct AWS account confirmed

📸 Screenshot:
<img width="1032" height="504" alt="image" src="https://github.com/user-attachments/assets/4e982204-50b9-4b62-9c55-fe33589e8ac5" />

### Step 2: Discover Latest Ubuntu 22.04 AMI
```
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text
```
#### Expected Outcome

`Latest Ubuntu AMI ID retrieved`

📸 Screenshot:
<img width="1032" height="671" alt="image" src="https://github.com/user-attachments/assets/18aaebac-3a33-4963-bdbe-2c9039e5763e" />

### Step 3: Launch EC2 Instance
```
aws ec2 run-instances \
  --image-id <ubuntu-ami-id> \
  --count 1 \
  --instance-type t2.micro \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=nautilus-ec2}]'
```
#### Expected Outcome

EC2 instance launched

Instance tagged as `nautilus-ec2`

📸 Screenshots:
<img width="1036" height="830" alt="image" src="https://github.com/user-attachments/assets/0e59112a-a29a-4ec1-8487-1afe5d85d9a5" />
<img width="1032" height="825" alt="image" src="https://github.com/user-attachments/assets/521d1d1c-97c2-46e0-9c6a-87db195b9fb9" />
<img width="1037" height="831" alt="image" src="https://github.com/user-attachments/assets/ff03585c-72da-48e9-8ce7-d96de01b2279" />
<img width="1030" height="739" alt="image" src="https://github.com/user-attachments/assets/13be1bcd-d154-4e93-b1da-f1d8ef4155a0" />

### Step 4: Retrieve SNS Topic ARN
```
aws sns list-topics \
  --query "Topics[?contains(TopicArn, 'nautilus-sns-topic')].TopicArn" \
  --output text
```
#### Expected Outcome

`SNS Topic ARN returned`

📸 Screenshot:
<img width="1035" height="340" alt="image" src="https://github.com/user-attachments/assets/0bdc588b-9ba4-47d6-b8db-d35684db0fb2" />

### Step 5: Create CloudWatch CPU Alarm
```
aws cloudwatch put-metric-alarm \
  --alarm-name "nautilus-alarm" \
  --alarm-description "Alarm when CPU exceeds 90 percent" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --evaluation-periods 1 \
  --alarm-actions <sns-topic-arn> \
  --unit Percent
```

#### Alarm Logic

- Metric: CPUUtilization

- Threshold: ≥ 90%

- Period: 5 minutes

- Evaluation: 1 datapoint

📸 Screenshot:
<img width="1039" height="550" alt="image" src="https://github.com/user-attachments/assets/6755463a-6c02-4807-8ff9-74eaf005a391" />

### Step 6: Validate Alarm Configuration
```
aws cloudwatch describe-alarms \
  --alarm-names "nautilus-alarm"
```

#### Expected Outcome
```
Alarm exists

State = OK

SNS action attached
```
📸 Screenshot:
<img width="1037" height="870" alt="image" src="https://github.com/user-attachments/assets/1c10d3b0-f196-44c7-9a6a-df8c85d67d69" />

### Step 7: Confirm EC2 Health Status
```
aws ec2 describe-instance-status \
  --instance-ids <instance-id>
```

#### Expected Outcome

- Instance state: `running`

- System & instance checks: `passed`

📸 Screenshot:
<img width="1027" height="856" alt="image" src="https://github.com/user-attachments/assets/30c07378-729f-4293-a968-27ad66cd6d29" />

## Final Outcome

- EC2 instance successfully deployed

- CPU utilization monitored via CloudWatch

- Alarm triggers on CPU ≥ 90%

- SNS notifications configured

- Instance health validated

## Optional Cleanup

- `aws cloudwatch delete-alarms --alarm-names nautilus-alarm`

- `aws ec2 terminate-instances --instance-ids <instance-id>`

## Design Notes

- Monitoring-first architecture

- CLI-driven reproducibility

- Threshold-based alerting

- Minimal assumptions

- Production-aligned observability pattern
