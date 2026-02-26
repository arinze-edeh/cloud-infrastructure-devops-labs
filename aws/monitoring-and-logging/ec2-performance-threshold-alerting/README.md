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
