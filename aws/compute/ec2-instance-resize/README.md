# EC2 Instance Right-Sizing: Controlled Downgrade for Cost Optimization

> **Classification:** Production Infrastructure Change  
> **Cloud Provider:** AWS  
> **Region:** us-east-1 (N. Virginia)  
> **Instance Modified:** `nautilus-ec2`  
> **Change Type:** EC2 Instance Type Downgrade (`t2.micro` to `t2.nano`)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Solution Summary](#solution-summary)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Step-by-Step Execution](#step-by-step-execution)
  - [Step 1: Authenticate into AWS Console](#step-1-authenticate-into-aws-console)
  - [Step 2: Locate the EC2 Instance](#step-2-locate-the-ec2-instance)
  - [Step 3: Verify Instance Health](#step-3-verify-instance-health)
  - [Step 4: Stop the EC2 Instance](#step-4-stop-the-ec2-instance)
  - [Step 5: Modify the Instance Type](#step-5-modify-the-instance-type)
  - [Step 6: Start the EC2 Instance](#step-6-start-the-ec2-instance)
  - [Step 7: Final Validation](#step-7-final-validation)
- [Outcome and Results](#outcome-and-results)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Operational Best Practices](#operational-best-practices)
- [Lessons Learned](#lessons-learned)
- [Skills Demonstrated](#skills-demonstrated)

---

## Overview

This document provides a production-grade walkthrough of an **EC2 instance right-sizing operation** performed by the Nautilus DevOps team. Following a routine infrastructure audit that surfaced an underutilized compute instance, a controlled downgrade from `t2.micro` to `t2.nano` was executed to reduce cloud spend while maintaining full service continuity.

This runbook is suitable for use as:
- **Onboarding documentation** for engineers new to AWS EC2 lifecycle operations
- **Production handoff documentation** for infrastructure change management
- **Reference material** for FinOps and cloud cost optimization workflows

---

## Problem Statement

During a periodic infrastructure audit, the `nautilus-ec2` instance was identified as consistently operating well below its provisioned compute capacity. The instance was running as a `t2.micro` (1 vCPU, 1 GB RAM) while actual workload telemetry indicated that a smaller instance class was sufficient.

**Impact of inaction:**
- Ongoing over-provisioning costs with no corresponding performance benefit
- Poor alignment between allocated resources and actual workload requirements
- Missed opportunity to demonstrate FinOps discipline within the engineering org

---

## Solution Summary

| Attribute | Before | After |
|---|---|---|
| Instance Type | `t2.micro` | `t2.nano` |
| vCPU | 1 | 1 |
| RAM | 1 GB | 0.5 GB |
| Instance State | Running | Running |
| Status Checks | 2/2 Passed | 2/2 Passed |

The change was executed using a **stop-modify-start** pattern, which is the standard AWS-supported method for changing the instance type of an EBS-backed EC2 instance.

---

## Architecture Context

- **Cloud Provider:** AWS
- **Region:** `us-east-1` (N. Virginia)
- **Instance Name:** `nautilus-ec2`
- **Instance ID:** `i-0bf04af755fcd4df5`
- **Availability Zone:** `us-east-1b`
- **Storage Type:** EBS-backed (required for instance type modification)
- **Network:** Default VPC (`vpc-0bc637648408579d8`), Private IP `172.31.30.142`

---

## Prerequisites

Before initiating this change, confirm the following preconditions are met:

- **Instance exists** in the target region with the name `nautilus-ec2`
- **Instance is EBS-backed** (instance store-backed instances cannot be resized using this method)
- **IAM permissions** are sufficient to stop, modify, and start EC2 instances
- **Status checks pass** (`2/2 checks passed`) before any modification is attempted
- **No active SSH sessions or critical processes** are running that cannot tolerate a controlled restart
- **Change window** is approved per your organization's change management process

---

## Step-by-Step Execution

### Step 1: Authenticate into AWS Console

**Intent:** Establish an authenticated session in the correct AWS account and region before performing any infrastructure changes. Region mismatch is one of the most common causes of failed or misdirected operations.

**Actions:**
- Open the AWS Management Console
- Log in using the provided IAM credentials
- Confirm the active region in the top-right navigation bar is set to **US East (N. Virginia) / us-east-1**

**Verification:** The region selector must display `United States (N. Virginia)` before proceeding.

![AWS Console Home with region selector open showing us-east-1 selected](https://github.com/user-attachments/assets/a9491e94-5d4f-4216-aca0-4bdef15b1af9)

*AWS Console Home with the region dropdown open, confirming `us-east-1` is the active region.*

---

### Step 2: Locate the EC2 Instance

**Intent:** Navigate directly to the target instance to confirm its existence, current state, and instance type prior to any modification.

**Actions:**
- Navigate to **EC2** via the Services menu or search bar
- Open **Instances** from the left navigation panel
- Filter by instance name `nautilus-ec2` or use the search bar to locate the instance

**Verification:** The instance `nautilus-ec2` appears in the list with `Instance state = Running` and `Instance type = t2.micro`.

![EC2 instance status checks showing 2/2 checks passed for nautilus-ec2](https://github.com/user-attachments/assets/34a63182-a9d2-44f3-9ba3-3b3db04b5127)

*EC2 Instances view showing `nautilus-ec2` in a `Running` state with `t2.micro` type confirmed.*

---

### Step 3: Verify Instance Health

**Intent:** Instance health validation is a mandatory gate before performing any lifecycle operation. Proceeding without confirmed status checks risks modifying an instance that is already in a degraded state, which can compound issues.

**Actions:**
- Select `nautilus-ec2` to open its detail panel
- Navigate to the **Status and alarms** tab or observe the **Status check** column in the instance list
- Confirm the status reads **2/2 checks passed**
- **Do not proceed** if the status is `Initializing` or shows a failed check

**Verification:** Both the **System status check** and **Instance status check** must show green with `2/2 checks passed`.

> **Operational Note:** If either check is failing, investigate using the EC2 System Log or Serial Console before attempting any state change. Stopping an already-degraded instance without diagnosis may result in data inconsistency.

![EC2 instance status checks showing 2/2 checks passed for nautilus-ec2](https://github.com/user-attachments/assets/34a63182-a9d2-44f3-9ba3-3b3db04b5127)

*Status checks for `nautilus-ec2` showing `2/2 checks passed`, confirming the instance is healthy and safe to modify.*

---

### Step 4: Stop the EC2 Instance

**Intent:** AWS requires an EBS-backed EC2 instance to be in a **stopped** state before its instance type can be changed. This is a hard platform constraint. Attempting to change the type of a running instance will be rejected.

**Actions:**
- Select the checkbox next to `nautilus-ec2`
- Click **Instance state** in the top action bar
- Select **Stop instance**
- Confirm the stop action in the modal dialog
- Wait until the **Instance state** column transitions from `Running` to `Stopped`

**Verification:** The instance state must read `Stopped` before proceeding to Step 5. The console will display a green success banner: *"Successfully initiated stopping of i-0bf04af755fcd4df5"*.

> **Risk Note:** Stopping an instance will terminate any in-memory data and active connections. Ensure no writes are in flight and that any application-level shutdown hooks have been accounted for before initiating the stop.

![EC2 instance showing Stopped state with success banner after stop initiation](https://github.com/user-attachments/assets/d3931cd9-eb7b-434f-a3d0-2f9cd51dcc58)

*Console confirms the stop was successfully initiated. The instance state now reads `Stopped`.*

---

### Step 5: Modify the Instance Type

**Intent:** With the instance fully stopped, the instance type attribute can now be safely changed. This operation modifies the compute profile of the instance without affecting attached EBS volumes, Elastic IPs, security groups, or any other associated resources.

**Actions:**
- Confirm `nautilus-ec2` is still selected
- Click **Actions** in the top action bar
- Navigate to **Instance settings** and select **Change instance type**
- In the dialog, select `t2.nano` from the instance type dropdown
- Click **Apply** to confirm the change

**Verification:** The console will display a green success banner: *"Instance type changed successfully"*. The **Instance type** column in the instances list will now reflect `t2.nano`.

> **Compatibility Note:** `t2.nano` is compatible with the same AMI and EBS volumes used by `t2.micro`. No OS-level reconfiguration is required for this downgrade. For other instance family changes (e.g., moving between x86 and ARM), AMI compatibility must be validated first.

![EC2 instance list showing nautilus-ec2 with instance type updated to t2.nano and Stopped state](https://github.com/user-attachments/assets/4705c764-34e5-4750-bacb-76455e4e63b4)

*Instance type successfully changed to `t2.nano`. The instance remains in a `Stopped` state, ready to be started.*

---

### Step 6: Start the EC2 Instance

**Intent:** With the instance type change applied, the instance is restarted to restore service. AWS will provision the instance on hardware matching the new `t2.nano` profile.

**Actions:**
- Confirm `nautilus-ec2` is still selected
- Click **Instance state** in the top action bar
- Select **Start instance**
- Monitor the **Instance state** column until it transitions from `Pending` to `Running`
- Observe the **Status check** column initializing

**Verification:** The console will display a green success banner: *"Successfully initiated starting of i-0bf04af755fcd4df5"*. The instance state will read `Running` and a new **Public IPv4 address** will be assigned.

> **IP Address Note:** When an EC2 instance without an Elastic IP is stopped and restarted, it will receive a **new public IPv4 address**. If your application or DNS records depend on a static public IP, an Elastic IP must be attached prior to stopping the instance. In this case, the new public IP assigned was `98.80.69.209`.

![EC2 instance showing Running state with new public IP after starting with t2.nano type](https://github.com/user-attachments/assets/172ffa53-09c9-42ec-b839-589f158e8408)

*`nautilus-ec2` successfully started. Instance state is `Running`, instance type is `t2.nano`, and a new public IPv4 address (`98.80.69.209`) has been assigned.*

---

### Step 7: Final Validation

**Intent:** A structured post-change validation confirms that all expected attributes are correct and the instance is operating normally under its new configuration. This step serves as the formal acceptance criteria for the change.

**Validation Checklist:**

| Attribute | Expected Value | Confirmed |
|---|---|---|
| Instance Name | `nautilus-ec2` | Yes |
| Instance ID | `i-0bf04af755fcd4df5` | Yes |
| Instance Type | `t2.nano` | Yes |
| Instance State | `Running` | Yes |
| Region | `us-east-1` | Yes |
| Availability Zone | `us-east-1b` | Yes |
| Status Checks | `2/2 checks passed` | Yes |
| Private IPv4 | `172.31.30.142` | Yes |

**Actions:**
- Select `nautilus-ec2` to open the detail panel
- Verify the **Instance summary** section reflects `t2.nano` as the instance type
- Confirm **Instance state** is `Running`
- Wait for **Status checks** to complete initialization and confirm `2/2 checks passed`

![EC2 instance details confirming nautilus-ec2 is Running with t2.nano instance type](https://github.com/user-attachments/assets/172ffa53-09c9-42ec-b839-589f158e8408)

*Final state of `nautilus-ec2`: instance type `t2.nano`, state `Running`, all attributes verified against expected values.*

---

## Outcome and Results

The right-sizing operation was completed successfully with zero data loss and no unplanned downtime beyond the controlled stop/start window.

**Summary:**
- EC2 instance `nautilus-ec2` was safely downgraded from `t2.micro` to `t2.nano`
- The instance passed all post-change health checks
- Service was restored without any configuration changes to the guest OS or application layer
- Cost reduction was achieved through a controlled, auditable infrastructure change

---

## Risks and Edge Cases

| Risk | Mitigation |
|---|---|
| **Public IP changes on restart** | Attach an Elastic IP before stopping if a stable public endpoint is required |
| **Instance store data loss** | Not applicable here (EBS-backed); however, instance store volumes are wiped on stop |
| **Application not starting cleanly** | Validate application startup scripts and systemd services after instance starts |
| **Insufficient RAM after downgrade** | Monitor memory usage post-change; t2.nano provides 0.5 GB which may constrain memory-intensive workloads |
| **Credit exhaustion on T2 burstable instances** | T2 instances use CPU credit model; monitor `CPUCreditBalance` in CloudWatch post-resize |
| **AMI incompatibility on family changes** | Validate AMI and driver compatibility when changing instance families (not applicable for t2 to t2) |

---

## Operational Best Practices

- **Always validate status checks** before stopping an instance. A degraded instance should be diagnosed before lifecycle operations are performed.
- **Document the change window** in your change management system before executing.
- **Use Elastic IPs** for any instance that must maintain a stable public endpoint across stop/start cycles.
- **Monitor CloudWatch metrics** after resizing to confirm the workload performs acceptably on the new instance type. Key metrics: `CPUUtilization`, `CPUCreditBalance`, `NetworkIn/Out`, `StatusCheckFailed`.
- **Test in non-production first** when right-sizing production workloads for the first time on a given application stack.
- **Automate recurrence** using AWS Compute Optimizer and Cost Explorer recommendations to build a continuous right-sizing feedback loop.

---

## Lessons Learned

- **Right-sizing is iterative.** The initial instance type selection is rarely optimal at steady state. Establishing a regular audit cadence ensures compute resources track actual workload needs.
- **Stop/start is a safe primitive.** The AWS stop-modify-start pattern for EBS-backed instances is reliable and reversible. Confidence in this operation comes from understanding that EBS volumes persist independently of instance state.
- **IP addressing must be planned.** The public IP change on restart is a common operational surprise. Engineering teams should treat Elastic IPs as a standard component of any internet-facing EC2 deployment.
- **Health checks are a hard gate.** Skipping pre-change status validation is a significant operational risk. Building this check into runbooks as a mandatory step improves change safety.

---

## Skills Demonstrated

- **EC2 Lifecycle Management** -- Controlled stop, type modification, and restart operations
- **Cloud Cost Optimization (FinOps)** -- Resource right-sizing based on utilization data
- **Change Safety and Operational Discipline** -- Pre-change health validation, structured runbook execution, and post-change verification
- **AWS Console Navigation** -- EC2 dashboard, instance management, and status monitoring
- **Production Documentation** -- Structured runbook authoring suitable for team onboarding and handoff



























# EC2 Instance Right-Sizing (Cost Optimization)

## Project Context
The Nautilus DevOps team identified an underutilized EC2 instance during an infrastructure audit.  
To optimize cost and resource utilization, the instance type was safely downgraded while ensuring service continuity.

This lab demonstrates **controlled EC2 lifecycle management**, **change safety**, and **cost-aware decision making**.

---

## Cloud Provider
AWS

## Region
us-east-1 (N. Virginia)

---

## Objective

GOAL:
-  Change EC2 instance type from t2.micro → t2.nano
-  Ensure instance remains healthy and running post-change
Preconditions
## REQUIREMENTS:
-  Instance exists with name "nautilus-ec2"
-  Status checks must pass before modification
-  Instance must be stopped before resizing

## Step 1: Authenticate into AWS Console
- OPEN AWS Console
- LOGIN using provided credentials
- VERIFY region = us-east-1

📸 Screenshot:
<img width="1741" height="945" alt="image" src="https://github.com/user-attachments/assets/a9491e94-5d4f-4216-aca0-4bdef15b1af9" />
AWS Console dashboard with region set to us-east-1

## Step 2: Locate EC2 Instance
- NAVIGATE to EC2 Dashboard
- OPEN Instances
- SEARCH for instance named "nautilus-ec2"

📸 Screenshot:
<img width="1761" height="915" alt="image" src="https://github.com/user-attachments/assets/d80b6ca9-1bfa-46ad-9aa0-9bb11c03664a" />
EC2 instance list showing nautilus-ec2

## Step 3: Verify Instance Health
- CHECK instance status
- WAIT until Status Checks = "2/2 checks passed"
- DO NOT proceed if status = initializing

📸 Screenshot:
<img width="1685" height="949" alt="image" src="https://github.com/user-attachments/assets/34a63182-a9d2-44f3-9ba3-3b3db04b5127" />
Instance status checks showing 2/2 passed

## Step 4: Stop EC2 Instance
- SELECT nautilus-ec2
- CHANGE instance state → Stop
- WAIT until instance state = stopped

📸 Screenshot:
<img width="1917" height="950" alt="image" src="https://github.com/user-attachments/assets/d3931cd9-eb7b-434f-a3d0-2f9cd51dcc58" />
Instance state showing "stopped"

## Step 5: Modify Instance Type
- OPEN Actions menu
- SELECT Instance settings → Change instance type
- SET instance type = t2.nano
- APPLY changes

📸 Screenshot:
<img width="1894" height="902" alt="image" src="https://github.com/user-attachments/assets/4705c764-34e5-4750-bacb-76455e4e63b4" />
Change instance type dialog with t2.nano selected

## Step 6: Start EC2 Instance
- SELECT nautilus-ec2
- CHANGE instance state → Start
- WAIT until instance state = running

📸 Screenshot:
<img width="1904" height="950" alt="image" src="https://github.com/user-attachments/assets/172ffa53-09c9-42ec-b839-589f158e8408" />
Instance state showing "running"

## Step 7: Final Validation

- VERIFY:
  -  Instance name = nautilus-ec2
  -  Instance type = t2.nano
  -  Instance state = running
  -  Region = us-east-1

📸 Screenshot:
<img width="1904" height="950" alt="image" src="https://github.com/user-attachments/assets/172ffa53-09c9-42ec-b839-589f158e8408" />
EC2 instance details showing t2.nano and running state

## Outcome
RESULT:
-  EC2 instance successfully right-sized
-  Cost optimized without service disruption

## Skills Demonstrated
SKILLS:
-  EC2 lifecycle management
-  Cloud cost optimization (FinOps basics)
-  Safe infrastructure change execution
-  AWS Console navigation

## Recruiter Notes
- This project reflects real-world DevOps practices:

  -  Production-safe instance modification

  -  Cost-awareness in cloud environments

  -  Operational discipline using health checks
