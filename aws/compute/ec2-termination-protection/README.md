# Enabling EC2 Termination Protection on a Running Instance

Preventing accidental deletion of critical compute infrastructure through AWS-native instance protection controls.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment Details](#environment-details)
- [Architecture and Logic](#architecture-and-logic)
- [Prerequisites](#prerequisites)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Log In to the AWS Management Console](#step-1-log-in-to-the-aws-management-console)
  - [Step 2: Select the Target Region](#step-2-select-the-target-region)
  - [Step 3: Navigate to EC2 Instances](#step-3-navigate-to-ec2-instances)
  - [Step 4: Locate and Select the Target Instance](#step-4-locate-and-select-the-target-instance)
  - [Step 5: Enable Termination Protection](#step-5-enable-termination-protection)
  - [Step 6: Verify Protection Status](#step-6-verify-protection-status)
- [Validation](#validation)
- [Key Decisions](#key-decisions)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Outcome](#outcome)
- [Tags](#tags)

---

## Overview

This document covers the end-to-end process of enabling **termination protection** on a running Amazon EC2 instance (`devops-ec2`) using the AWS Management Console. Termination protection is a lightweight but critical control that prevents an instance from being accidentally deleted via the console, CLI, or API, adding a mandatory confirmation barrier before any termination can proceed.

This configuration is a foundational infrastructure hygiene task that belongs in every production deployment checklist.

---

## Problem Statement

EC2 instances powering production workloads, long-running batch jobs, or stateful services can be accidentally terminated by operators with broad IAM permissions, automation scripts, or misconfigured lifecycle policies. Without termination protection, a single misclick or erroneous API call can cause irreversible data loss and service disruption.

**Goal:** Apply a termination protection flag to an existing running instance to prevent accidental deletion, with zero downtime and no instance restart required.

---

## Environment Details

| Parameter | Value |
|---|---|
| Cloud Provider | AWS |
| Service | Amazon EC2 |
| Region | US East (N. Virginia) `us-east-1` |
| Availability Zone | `us-east-1a` |
| Instance Name | `devops-ec2` |
| Instance ID | `i-054dd1d9bc74e624a` |
| Instance Type | `t2.micro` |
| AMI | `al2023-ami-2023.4.20240319.1-kernel-6.1-x86_64` |
| Platform | Linux/UNIX |
| Public IPv4 | `3.84.21.142` |
| Private IPv4 | `172.31.88.242` |
| Action | Enable Termination Protection |

---

## Architecture and Logic

```
LOGIN to AWS Console
      |
      v
SELECT Region: us-east-1
      |
      v
NAVIGATE to EC2 > Instances
      |
      v
IDENTIFY target instance: devops-ec2
      |
      v
Actions > Instance Settings > Change Termination Protection
      |
      v
ENABLE Termination Protection (checkbox: enabled)
      |
      v
SAVE configuration
      |
      v
VERIFY: Instance Details panel shows "Termination protection: Enabled"
      |
      v
CONFIRM: Success banner displayed
```

---

## Prerequisites

- AWS Management Console access with sufficient IAM permissions:
  - `ec2:ModifyInstanceAttribute` on the target instance
- An existing running EC2 instance in the target region
- Region awareness: confirm you are operating in the correct region before making changes

---

## Step-by-Step Implementation

### Step 1: Log In to the AWS Management Console

Navigate to [https://console.aws.amazon.com](https://console.aws.amazon.com) and authenticate with your IAM credentials.

On the **Console Home**, confirm the account name in the top-right corner matches your target environment. This prevents accidental changes to wrong accounts in multi-account organizations.

> **Note:** The console displays several `Access denied` notices for `servicecatalog:ListApplications` and billing widgets. These are permission-scoped restrictions on the IAM user and do not impact EC2 operations.

**Screenshot: AWS Console Home**

<img width="1735" height="942" alt="AWS Console Home showing account context and service widgets" src="https://github.com/user-attachments/assets/392848b2-e3f9-4ce2-88ed-86b20ebb5a0f" />

---

### Step 2: Select the Target Region

Click the **region selector** in the top-right navigation bar. From the dropdown, select **US East (N. Virginia) `us-east-1`**, which is highlighted as the current region.

Region selection is critical. EC2 resources are region-scoped; selecting the wrong region will display an empty or unrelated instance list.

**Screenshot: Region selector with `us-east-1` highlighted**

<img width="1764" height="948" alt="AWS region selector dropdown with us-east-1 selected" src="https://github.com/user-attachments/assets/851eb819-a983-424c-9e5b-9cfe03b7f445" />

---

### Step 3: Navigate to EC2 Instances

From the search bar or the **Services** menu, navigate to **EC2**. In the left sidebar, under **Instances**, click **Instances**.

The instance list loads with an active filter `Instance state = running`, displaying one result: `devops-ec2`.

Confirm the following before proceeding:

- Instance state: **Running**
- Status checks: **2/2 checks passed**
- Availability Zone: **us-east-1a**
- Instance type: **t2.micro**

**Screenshot: EC2 Instances list showing `devops-ec2` in Running state**

<img width="1763" height="946" alt="EC2 Instances list with devops-ec2 running in us-east-1a" src="https://github.com/user-attachments/assets/2e1bd221-5380-4a9f-85b3-47409ae0f184" />

---

### Step 4: Locate and Select the Target Instance

Select the checkbox next to **devops-ec2** (`i-054dd1d9bc74e624a`). The instance detail panel opens in the lower pane.

To initiate the termination protection change, click **Actions** in the top-right toolbar, then navigate to:

`Actions > Instance Settings > Change Termination Protection`

This opens the **Change termination (deletion) protection** modal dialog.

**Screenshot: Instance selected with the termination protection modal open**

<img width="1774" height="945" alt="EC2 instance selected and termination protection dialog opened" src="https://github.com/user-attachments/assets/b975b96d-3dd5-4971-8f50-cef6eadf7876" />

---

### Step 5: Enable Termination Protection

In the **Change termination (deletion) protection** dialog:

- **Instance ID** displayed: `i-054dd1d9bc74e624a (devops-ec2)`
- **Termination protection** checkbox: **check the Enable box**

Click **Save** to apply the configuration.

> **What this does:** This sets the `disableApiTermination` attribute on the instance to `true`. Any subsequent termination request via the console, AWS CLI (`aws ec2 terminate-instances`), or API will be rejected with an error until this flag is explicitly disabled again.

**Screenshot: Termination protection dialog with Enable checkbox selected**

<img width="1775" height="943" alt="Termination protection modal with Enable checked and Save button visible" src="https://github.com/user-attachments/assets/9d09b366-7c5e-4d08-bde2-cb69bab88cce" />

---

### Step 6: Verify Protection Status

After saving, the console displays a **green success banner** at the top of the Instances page:

> *"Successfully enabled termination protection for instance i-054dd1d9bc74e624a. The instance can't be deleted."*

Scroll down in the instance detail panel to the **Instance details** section and confirm:

- **Termination protection:** `Enabled`
- **Stop protection:** `Disabled`
- **Instance state:** `Running` (no interruption occurred)

**Screenshot: Success banner and instance details confirming Termination protection: Enabled**

<img width="1791" height="946" alt="Success banner and instance details panel showing termination protection enabled" src="https://github.com/user-attachments/assets/bdb09f3a-edf7-4be6-88d4-6a6dbf2dd8db" />

---

## Validation

| Check | Expected Result | Observed |
|---|---|---|
| Console success banner displayed | Green banner with instance ID | Confirmed |
| Instance state post-change | Running (no restart) | Confirmed |
| Termination protection field | Enabled | Confirmed |
| Status checks | 2/2 checks passed | Confirmed |
| Instance accessible | Public IP reachable | Confirmed |

To validate via CLI after the fact:

```bash
aws ec2 describe-instance-attribute \
  --instance-id i-054dd1d9bc74e624a \
  --attribute disableApiTermination \
  --region us-east-1
```

Expected output:

```json
{
    "InstanceId": "i-054dd1d9bc74e624a",
    "DisableApiTermination": {
        "Value": true
    }
}
```

---

## Key Decisions

**Console over CLI for this task**
This change was applied through the AWS Management Console to match the documented workflow and provide visible confirmation dialogs. For automation at scale, `aws ec2 modify-instance-attribute --no-disable-api-termination` / `--disable-api-termination` is the preferred approach and can be scripted into provisioning pipelines.

**Termination protection only, not stop protection**
Stop protection was intentionally left disabled. Stopping the instance (for resizing, patching, or snapshotting) should remain a routine operation. Termination protection addresses a narrower and more permanent risk: permanent deletion of the instance.

**Applied to a running instance**
Termination protection can be toggled at any point in an instance's lifecycle, including while running, without triggering a reboot. This makes it safe to apply retroactively to production instances.

---

## Risks and Edge Cases

- **Termination protection does not prevent stop or reboot.** An operator can still stop or restart the instance freely.
- **Auto Scaling Groups bypass this flag.** If the instance is part of an ASG and the group decrements capacity, the ASG lifecycle process can terminate the instance regardless of the `disableApiTermination` flag. Protect ASG-managed instances at the group policy level, not per-instance.
- **IAM users with `ec2:ModifyInstanceAttribute` can disable protection.** This control is not a substitute for IAM least-privilege policies. If an operator can enable protection, they can also disable it. Restrict this permission to privileged roles.
- **CloudFormation stack deletion will fail** on resources with termination protection enabled. Account for this in IaC teardown workflows by first disabling protection or using `DeletionPolicy: Retain`.
- **Multi-region awareness:** Termination protection is instance-specific and region-scoped. Applying it in one region does not affect instances in other regions.

---

## Best Practices and Operational Considerations

- **Apply termination protection as a default for all production instances** during launch, either via the EC2 launch wizard, AWS CLI `--disable-api-termination`, or CloudFormation `DisableApiTermination: true`.
- **Combine with IAM policies** that restrict `ec2:TerminateInstances` to specific roles (e.g., a `BreakGlass` admin role), adding a second layer of access control.
- **Tag protected instances** with a metadata tag such as `TerminationProtection: enabled` to make their status discoverable through tag-based resource searches and AWS Config rules.
- **Audit using AWS Config:** The managed rule `ec2-instance-no-public-ip` and custom rules can be extended to flag instances lacking termination protection in production environments.
- **Review protection status during change windows** so operators are not surprised by failed termination attempts during planned decommissions.
- **Stop protection** (a separate attribute) should be considered for instances where even stopping could cause data corruption (e.g., in-memory caching workloads).

---

## Lessons Learned

- **Termination protection is a zero-downtime change.** Applying it to a running instance has no impact on instance state, network connectivity, or running workloads.
- **It is not enabled by default.** Every new EC2 instance launches without termination protection unless explicitly configured. This must be enforced through launch templates, IaC, or Service Control Policies (SCPs) at the organization level.
- **The console provides clear visual confirmation.** The success banner and the `Enabled` status in the Instance details panel make it straightforward to verify the change without requiring CLI access.
- **Access denied errors on the console home are expected in restricted IAM environments** (e.g., `servicecatalog:ListApplications`, billing widgets) and do not indicate a problem with EC2 permissions. Always scope troubleshooting to the specific action being performed.

---

## Outcome

- Termination protection successfully enabled on `devops-ec2` (`i-054dd1d9bc74e624a`) in `us-east-1`
- Instance remained in `Running` state with no service interruption
- `DisableApiTermination` attribute set to `true`
- Console success banner confirmed the change
- Instance detail panel reflects `Termination protection: Enabled`

---

## Tags

`aws` `ec2` `compute` `cloud-security` `infrastructure` `instance-protection` `devops` `production-hardening`






























# Enable EC2 Termination Protection

## Lab Overview
- This lab demonstrates how to enable **termination protection** on an existing Amazon EC2 instance using the AWS Management Console.  
- Termination protection prevents accidental deletion of critical compute resources.

---

## Environment Details

| Item | Value |
|-----|------|
| Cloud Provider | AWS |
| Service | EC2 |
| Region | us-east-1 |
| Instance Name | devops-ec2 |
| Action | Enable Termination Protection |

---

## Objective

- Locate an existing EC2 instance
- Enable termination protection
- Verify protection status

---

## High-Level Logic

- LOGIN to AWS Console
- SELECT correct AWS region

- NAVIGATE to EC2 instances
- IDENTIFY target instance

- IF termination protection is disabled:
  -  ENABLE termination protection

- VERIFY configuration
- CONFIRM success

## 🛠️ Step-by-Step Implementation

## Step 1: AWS Console Login

📸 Screenshot:
<img width="1735" height="942" alt="image" src="https://github.com/user-attachments/assets/392848b2-e3f9-4ce2-88ed-86b20ebb5a0f" />

## Step 2: Select Region (us-east-1)
📸 Screenshot:
<img width="1764" height="948" alt="image" src="https://github.com/user-attachments/assets/851eb819-a983-424c-9e5b-9cfe03b7f445" />

## Step 3: Open EC2 Dashboard
📸 Screenshot:
<img width="1763" height="946" alt="image" src="https://github.com/user-attachments/assets/2e1bd221-5380-4a9f-85b3-47409ae0f184" />

## Step 4: Locate EC2 Instance
📸 Screenshot:
<img width="1774" height="945" alt="image" src="https://github.com/user-attachments/assets/b975b96d-3dd5-4971-8f50-cef6eadf7876" />

## Step 5: Enable Termination Protection
📸 Screenshot:
<img width="1775" height="943" alt="image" src="https://github.com/user-attachments/assets/9d09b366-7c5e-4d08-bde2-cb69bab88cce" />

## Step 6: Verify Protection Status
📸 Screenshot:
<img width="1791" height="946" alt="image" src="https://github.com/user-attachments/assets/bdb09f3a-edf7-4be6-88d4-6a6dbf2dd8db" />

## ✅ Outcome
- EC2 termination protection enabled
- Accidental termination prevented
- Task completed successfully

## Key AWS Concepts Demonstrated
- EC2 instance management

- AWS region awareness

- Instance safety controls

- Infrastructure protection best practices

## Tags
`aws` `ec2` `compute` `cloud-security` `infrastructure`









