# [AWS EC2 Instance Protection: Enabling Stop Protection via the Management Console](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> Preventing accidental instance termination for production-critical workloads on Amazon EC2 using the AWS Management Console.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Lab Environment](#lab-environment)
- [Implementation](#implementation)
  - [Step 1: Log In and Confirm Region](#step-1-log-in-and-confirm-region)
  - [Step 2: Navigate to EC2](#step-2-navigate-to-ec2)
  - [Step 3: Locate and Select the Target Instance](#step-3-locate-and-select-the-target-instance)
  - [Step 4: Open Instance Details](#step-4-open-instance-details)
  - [Step 5: Enable Stop Protection](#step-5-enable-stop-protection)
  - [Step 6: Confirm and Save](#step-6-confirm-and-save)
  - [Step 7: Validate the Configuration](#step-7-validate-the-configuration)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Lessons Learned](#lessons-learned)
- [Final Outcome](#final-outcome)

---

## Overview

This lab documents the process of enabling **Stop Protection** on an Amazon EC2 instance using the AWS Management Console. Stop Protection is an instance-level safeguard that blocks stop actions issued through the console, CLI, or API, preventing accidental shutdown of workloads that must remain available.

This procedure was executed as part of an infrastructure migration workflow, where protecting running compute resources from inadvertent human or automation-triggered stops is a critical operational requirement.

---

## Problem Statement

During active infrastructure migration and day-to-day operations, EC2 instances supporting production workloads are at risk of being accidentally stopped. This can occur through:

- Misidentified instances during bulk operations
- Operator error in the console or CLI
- Automation scripts with overly broad instance selectors

Without a protection mechanism, these events can cause unplanned downtime. **Stop Protection** addresses this by requiring the feature to be explicitly disabled before any stop action can succeed, introducing a deliberate confirmation checkpoint.

---

## Architecture Context

**Cloud Provider:** Amazon Web Services (AWS)
**Service:** Amazon EC2
**Region:** `us-east-1` (United States, N. Virginia)
**Instance Name:** `datacenter-ec2`
**Instance ID:** `i-0c19555246b823f26`
**Instance Type:** `t2.micro`
**AMI:** Amazon Linux 2023 (`al2023-ami-2023.4.20240319.1-kernel-6.1-x86_64`)
**Platform:** Linux/UNIX
**Availability Zone:** `us-east-1d`
**Public IPv4:** `54.82.93.57`
**Private IPv4:** `172.31.16.169`

---

## Prerequisites

- Active AWS account with console access (KodeKloud lab environment used here)
- IAM permissions to modify EC2 instance attributes (`ec2:ModifyInstanceAttribute`)
- An existing EC2 instance in a Running state
- Browser access to the AWS Management Console

---

## Lab Environment

This lab was executed inside a KodeKloud-provisioned AWS sandbox account. The EC2 instance `datacenter-ec2` was pre-running at lab start. No instance provisioning was required. The objective was exclusively the configuration of Stop Protection on the existing instance.

---

## Implementation

### Step 1: Log In and Confirm Region

Log in to the AWS Management Console using the provided credentials. Before proceeding, verify that the active region is **us-east-1 (N. Virginia)**. The region selector is visible in the top-right corner of the console header.

Ensuring the correct region is set before any action eliminates the risk of modifying the wrong instance in a multi-region account.

**Screenshot:** AWS Console Home with the region selector open confirming `us-east-1` is active.

<img width="1750" height="946" alt="AWS Console Dashboard showing region set to us-east-1" src="https://github.com/user-attachments/assets/4b1604ea-0fce-4ccf-a45a-eee6fefbfa9e" />

---

### Step 2: Navigate to EC2

From the Console Home, open the **Services** menu and select **EC2**, or use the search bar (Alt+S) and type `EC2`. This opens the EC2 service dashboard.

On the EC2 Dashboard, confirm that **Instances (running): 1** is displayed in the Resources panel. This confirms the target instance is active in the current region before navigating deeper.

**Screenshot:** EC2 service dashboard showing one running instance and the resource summary panel.

<img width="1831" height="946" alt="EC2 service dashboard confirming one running instance" src="https://github.com/user-attachments/assets/ac2f1de9-3452-40dc-8e01-f03fb34de002" />

---

### Step 3: Locate and Select the Target Instance

In the left navigation panel, click **Instances** under the **Instances** section. The Instances list will load, filtered by default to show running instances.

Locate the instance named **datacenter-ec2**. Confirm the following attributes match the expected values before proceeding:

- **Instance ID:** `i-0c19555246b823f26`
- **Instance state:** Running
- **Instance type:** t2.micro
- **Status check:** 2/2 checks passed
- **Availability Zone:** us-east-1d

**Screenshot:** EC2 Instances list showing `datacenter-ec2` with `Running` state and passing status checks.

<img width="1780" height="946" alt="EC2 instances list showing datacenter-ec2 in running state" src="https://github.com/user-attachments/assets/22d1cd0d-eaf3-4634-9c60-2cd99bc09d7f" />

---

### Step 4: Open Instance Details

Click the checkbox next to **datacenter-ec2** to select it. The instance detail pane will expand at the bottom of the screen, displaying the **Details**, **Status and alarms**, **Monitoring**, **Security**, **Networking**, **Storage**, and **Tags** tabs.

Confirm the following from the **Details** tab before making changes:

- **Public IPv4 address:** `54.82.93.57`
- **Private IPv4 address:** `172.31.16.169`
- **Instance state:** Running
- **Instance type:** t2.micro

**Screenshot:** Instance selected with the detail pane expanded showing the Instance Summary.

<img width="1776" height="948" alt="Instance detail pane showing datacenter-ec2 selected with full summary visible" src="https://github.com/user-attachments/assets/1d1a6ede-3107-4a64-928a-5689d64c279d" />

---

### Step 5: Enable Stop Protection

With the instance selected, click the **Actions** button in the top-right toolbar. Navigate to:

**Actions** > **Instance settings** > **Change stop protection**

This opens the **Change stop protection** dialog box.

**Screenshot:** The "Change stop protection" dialog displaying the Instance ID and the Stop Protection checkbox.

<img width="1795" height="948" alt="Change stop protection dialog for datacenter-ec2" src="https://github.com/user-attachments/assets/efbec68f-7642-4606-ae5a-bc2cf83860da" />

---

### Step 6: Confirm and Save

In the **Change stop protection** dialog:

1. Verify the **Instance ID** matches: `i-0c19555246b823f26 (datacenter-ec2)`
2. Under **Stop protection**, check the **Enable** checkbox
3. Click **Save**

The dialog will close and a green success banner will appear at the top of the Instances page:

> **Enabled stop protection for i-0c19555246b823f26**

**Screenshot:** Success confirmation banner displayed after enabling Stop Protection.

<img width="1794" height="947" alt="Success banner confirming stop protection enabled for i-0c19555246b823f26" src="https://github.com/user-attachments/assets/5e17501a-8a22-40ef-94ac-814036bfef41" />

<img width="1798" height="942" alt="Instances page with green success banner and instance still in running state" src="https://github.com/user-attachments/assets/b31f144f-ec53-46fd-8183-a731e3cb71a6" />

---

### Step 7: Validate the Configuration

With the instance still selected, scroll down in the **Details** tab to the second section of the Instance Summary. Locate the **Stop protection** field and confirm the value reads:

> **Stop protection: Enabled**

Also note:
- **Termination protection:** Disabled (distinct from Stop Protection -- this controls termination, not stop actions)
- **Instance auto-recovery:** Default
- **Monitoring:** Disabled

This section provides the authoritative confirmation that the protection attribute was applied successfully at the instance level.

**Screenshot:** Instance Details section showing `Stop protection: Enabled` alongside other instance attributes.

<img width="1798" height="942" alt="Instance details confirming Stop protection is Enabled for datacenter-ec2" src="https://github.com/user-attachments/assets/b31f144f-ec53-46fd-8183-a731e3cb71a6" />

---

## Key Decisions

**Stop Protection vs. Termination Protection:** This lab enables Stop Protection only. These are two independent flags. Stop Protection prevents the instance from being stopped while still running; it does not prevent termination. For production workloads requiring full protection, both flags should be evaluated and enabled independently based on the use case.

**Console-only approach:** Stop Protection was configured via the Management Console in this lab to demonstrate the UI workflow. The same result can be achieved via the AWS CLI using:

```bash
aws ec2 modify-instance-attribute \
  --instance-id i-0c19555246b823f26 \
  --no-disable-api-stop
```

For infrastructure-as-code environments, this attribute should be managed in Terraform using the `disable_api_stop = false` argument inside the `aws_instance` resource to ensure the protection is codified and version-controlled.

---

## Best Practices and Operational Considerations

- **Apply Stop Protection to all production instances** as a baseline policy, not just critical ones. The cost of enabling it is zero; the cost of an accidental stop in production can be significant.
- **Pair with Termination Protection** for instances that should never be destroyed without explicit intervention. The two flags operate independently.
- **Use IAM policies** to restrict who can disable Stop Protection. Limit the `ec2:ModifyInstanceAttribute` permission to senior engineers or break-glass accounts in production environments.
- **Tag protected instances** with a metadata tag such as `StopProtection: enabled` to make protection state queryable via resource inventories and AWS Config rules.
- **Audit periodically** using AWS Config or a Lambda-based compliance check to ensure protection flags have not been silently removed during instance lifecycle events or automation runs.
- **Document in runbooks** the exact procedure to disable Stop Protection before any planned maintenance window, to avoid operational confusion during incident response.

---

## Risks and Edge Cases

- **Automation failures:** Runbooks or automation scripts that attempt to stop protected instances will receive an `OperationNotPermitted` API error. Ensure pipelines that manage instance state handle this error explicitly.
- **Silent protection bypass:** Stop Protection does not prevent AWS-initiated stop events such as scheduled retirement or Spot interruptions (for Spot Instances). It only blocks user and API-initiated stops.
- **Instance type changes:** Some instance type changes require a stop. If Stop Protection is enabled, the stop required for the type change will be blocked. Disable Stop Protection, complete the modification, then re-enable it.
- **Lab environment resets:** In sandbox environments, Stop Protection may be removed when lab sessions reset or instances are reprovisioned. Always verify protection state at the start of each lab session if continuity is required.

---

## Lessons Learned

- Stop Protection is a lightweight but highly effective safeguard that requires no additional cost or infrastructure overhead.
- The AWS Console provides clear visual confirmation of protection state in the Instance Details panel, making manual auditing straightforward.
- Stop Protection and Termination Protection serve distinct purposes and should be treated as complementary controls rather than interchangeable ones.
- In migration projects, applying Stop Protection immediately after provisioning a critical instance, before any workload is deployed, is the correct operational sequence rather than retrofitting it later.

---

## Final Outcome

| Attribute | Value |
|---|---|
| Instance Name | datacenter-ec2 |
| Instance ID | i-0c19555246b823f26 |
| Region | us-east-1 |
| Instance State | Running |
| Stop Protection | **Enabled** |
| Termination Protection | Disabled |
| Validation Method | EC2 Console, Instance Details tab |

Stop Protection was successfully enabled on `datacenter-ec2`. The instance is now protected from accidental stop actions issued through the AWS Management Console, CLI, or API. The configuration was confirmed via the Instance Details panel, with the `Stop protection: Enabled` field visible in the instance attribute summary.






























# AWS EC2 – Enable Stop Protection (Management Console)

## Lab Overview
As part of an infrastructure migration, this lab demonstrates how to enable **Stop Protection** for an existing Amazon EC2 instance using the **AWS Management Console**.

Stop Protection prevents an EC2 instance from being accidentally stopped through the console, CLI, or API.

---

## Cloud Provider
- **Platform:** Amazon Web Services (AWS)
- **Service:** EC2
- **Region:** us-east-1

---

## Requirements
- Existing EC2 instance named `datacenter-ec2`
- AWS Console access
- Permissions to modify EC2 instance settings

---

## Task Objective
Enable **Stop Protection** for the EC2 instance:
- **Instance Name:** datacenter-ec2
- **Region:** us-east-1

---

## Step-by-Step Implementation (AWS Console)

### Step 1: Log in to AWS Console
- Open the AWS Console URL
- Sign in with the provided credentials
- Ensure the region is set to **us-east-1**

📸 **Screenshot:**  
`AWS Console Dashboard showing region set to us-east-1`

<img width="1750" height="946" alt="image" src="https://github.com/user-attachments/assets/4b1604ea-0fce-4ccf-a45a-eee6fefbfa9e" />

---

### Step 2: Navigate to EC2 Service
- Open **Services** → **EC2**
- Click **Instances** from the left navigation panel

📸 **Screenshot:**  
`EC2 service dashboard`

<img width="1831" height="946" alt="image" src="https://github.com/user-attachments/assets/ac2f1de9-3452-40dc-8e01-f03fb34de002" />

---

### Step 3: Select the EC2 Instance
- Locate the instance named **datacenter-ec2**
- Select the instance by checking the box

📸 **Screenshot:**  
`EC2 instances list showing datacenter-ec2 selected`

<img width="1780" height="946" alt="image" src="https://github.com/user-attachments/assets/22d1cd0d-eaf3-4634-9c60-2cd99bc09d7f" />
<img width="1776" height="948" alt="image" src="https://github.com/user-attachments/assets/1d1a6ede-3107-4a64-928a-5689d64c279d" />

---

### Step 4: Enable Stop Protection
- Click **Actions**
- Navigate to **Instance settings**
- Select **Change stop protection**
- Check **Enable stop protection**
- Click **Save**

📸 **Screenshot:**  
`Change stop protection dialog with enable option selected`

<img width="1795" height="948" alt="image" src="https://github.com/user-attachments/assets/efbec68f-7642-4606-ae5a-bc2cf83860da" />
<img width="1794" height="947" alt="image" src="https://github.com/user-attachments/assets/5e17501a-8a22-40ef-94ac-814036bfef41" />

---

### Step 5: Verify Configuration
- Select the instance
- Open the **Details** tab
- Confirm **Stop protection: Enabled**

📸 **Screenshot:**  
`Instance details showing stop protection enabled`

<img width="1798" height="942" alt="image" src="https://github.com/user-attachments/assets/b31f144f-ec53-46fd-8183-a731e3cb71a6" />

---

## Final Outcome
- Stop Protection successfully enabled
- Instance cannot be stopped accidentally
- Configuration validated via EC2 instance details

---

## Key Takeaways
- Stop Protection adds a safety layer for critical workloads
- Configuration can be managed directly from the AWS Console
- Useful for production or sensitive environments
