# AWS EC2 Instance Termination: Infrastructure Cleanup (us-east-1)

> **Classification:** Infrastructure Operations | Cloud Cleanup  
> **Severity:** Low | **Impact:** Single Instance Decommission  
> **Region:** `us-east-1` (US East (N. Virginia))  
> **Outcome:** Confirmed `terminated` state for EC2 instance `xfusion-ec2`

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Scope & Impact Assessment](#2-scope--impact-assessment)
3. [Prerequisites & Access Requirements](#3-prerequisites--access-requirements)
4. [Risk Analysis & Considerations](#4-risk-analysis--considerations)
5. [Implementation Runbook](#5-implementation-runbook)
   - [Step 1: Authenticate to AWS Console](#step-1--authenticate-to-aws-console)
   - [Step 2: Validate & Set Target Region](#step-2--validate--set-target-region)
   - [Step 3: Navigate to EC2 Service Dashboard](#step-3--navigate-to-ec2-service-dashboard)
   - [Step 4: Locate Target Instance](#step-4--locate-target-instance)
   - [Step 5: Terminate the Instance](#step-5--terminate-the-instance)
   - [Step 6: Verify Termination State](#step-6--verify-termination-state)
6. [Final Verification Summary](#6-final-verification-summary)
7. [Troubleshooting & Edge Cases](#7-troubleshooting--edge-cases)
8. [Lessons Learned & Best Practices](#8-lessons-learned--best-practices)
9. [Tags & Metadata](#9-tags--metadata)

---

## 1. Problem Statement

During a planned infrastructure migration, an audit identified EC2 resources that are no longer required. The instance `xfusion-ec2` (type: `t2.micro`, region: `us-east-1`) has been decommissioned from active workloads and must be permanently deleted to:

- Eliminate unnecessary compute costs
- Reduce attack surface from unused, potentially unpatched instances
- Maintain a clean and auditable cloud inventory
- Comply with infrastructure hygiene standards expected in production environments

This runbook documents the end-to-end process: identification, termination, and verified confirmation of the `terminated` state.

---

## 2. Scope & Impact Assessment

| Attribute           | Value                          |
|---------------------|-------------------------------|
| Instance Name       | `xfusion-ec2`                 |
| Instance ID         | `i-0d87b86ea4f037cec`         |
| Instance Type       | `t2.micro`                    |
| Region              | `us-east-1` (N. Virginia)     |
| Availability Zone   | `us-east-1a`                  |
| Public IPv4         | `3.93.200.72`                 |
| Private IPv4        | `172.31.26.42`                |
| Initial State       | `Running`                     |
| Target State        | `Terminated`                  |
| Downstream Impact   | None (confirmed decommissioned) |

> **Important:** EC2 instance termination is **irreversible**. Once in the `terminated` state, the instance and its ephemeral storage cannot be recovered. Verify there are no active dependencies (load balancers, Elastic IPs, attached EBS volumes with critical data) before proceeding.

---

## 3. Prerequisites & Access Requirements

Before executing this runbook, confirm the following:

- **IAM Permissions:** The operator account must have the `ec2:TerminateInstances` permission scoped to the target region. Verify using IAM Policy Simulator if uncertain.
- **Region Access:** Confirm `us-east-1` is enabled for the account (visible in the region selector dropdown).
- **Data Backup:** Any persistent data on attached EBS volumes must be snapshotted or backed up if retention is required. Termination will delete non-preserved root volumes.
- **Dependency Check:** Ensure the instance is not referenced by Auto Scaling Groups, Elastic Load Balancers, Route 53 health checks, or other managed services.
- **Change Approval:** Obtain change management approval per your organization's ITSM process (e.g., JIRA, ServiceNow) before executing in production.

---

## 4. Risk Analysis & Considerations

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Terminating the wrong instance | Low | Verify Instance ID, Name tag, and region before confirming |
| Data loss from unsnapshotted EBS volumes | Medium | Snapshot EBS volumes prior to termination if data retention is needed |
| Instance protected against termination | Low | Check "Termination Protection" attribute; disable if enabled |
| IAM permission denial | Low | Verify `ec2:TerminateInstances` and `ec2:DescribeInstances` are granted |
| Instance part of an Auto Scaling Group | Medium | Verify ASG association; terminate via ASG, not directly, if applicable |

> **Note on Termination Protection:** AWS allows enabling termination protection on instances. If the `Terminate` option is grayed out in the console, navigate to **Actions → Instance Settings → Change Termination Protection** and disable it first.

---

## 5. Implementation Runbook

### Step 1: Authenticate to AWS Console

**Objective:** Establish an authenticated session with the appropriate IAM user or role.

**Actions:**
1. Open a browser and navigate to [https://console.aws.amazon.com](https://console.aws.amazon.com)
2. Log in using your IAM credentials (username/password) or SSO provider
3. Confirm the correct account is loaded by verifying the account ID shown in the top-right navigation bar

**Operational Note:** Use MFA-enabled accounts for all production operations. If your organization uses AWS SSO (IAM Identity Center), authenticate through the Access Portal to assume the appropriate role.

**Screenshot: Console Home (Post-Login):**

![Step 1: AWS Console Home](./screenshots/step1_console_home.png)

> The console home displays "No recently visited services," confirming this is a fresh session. Several widgets show "Access denied," which is expected given the scoped IAM policy applied to this lab account; it does not impact the ability to perform EC2 operations.

---

### Step 2: Validate & Set Target Region

**Objective:** Confirm the AWS session is scoped to `us-east-1` (US East (N. Virginia)) where the target instance resides.

**Actions:**
1. Locate the **region selector** in the top-right corner of the navigation bar
2. Click to expand the dropdown
3. Verify or select **US East (N. Virginia): `us-east-1`**

**Why This Matters:** AWS EC2 instances are region-scoped resources. Operating in the wrong region will result in the target instance not appearing in the instance list, potentially leading to confusion or action on the wrong resource.

**Screenshot: Region Selector Dropdown:**

![Step 2: Region Selection](./screenshots/step2_region_selector.png)

> `us-east-1` is highlighted as the Current Region. No region change was required; the session was already correctly scoped.

---

### Step 3: Navigate to EC2 Service Dashboard

**Objective:** Access the EC2 service console to view compute resources in the target region.

**Actions:**
1. From the AWS Console Home, click **Services** (top-left grid icon) or use the search bar
2. Type `EC2` and select **EC2** from the results
3. Confirm the EC2 Dashboard loads and displays the correct region (`United States (N. Virginia)`)

**Dashboard Validation:**
- **Instances (running): 1**: confirms the target instance is active
- **Volumes: 1**: indicates an EBS volume is attached (verify retention requirements)
- **Security groups: 1**: note for cleanup if no other resources reference it

**Screenshot: EC2 Dashboard:**

![Step 3: EC2 Dashboard](./screenshots/step3_ec2_dashboard.png)

> The dashboard confirms 1 running instance in `us-east-1`. Some widgets display API errors (e.g., Dedicated Hosts, Capacity Reservations) due to restricted IAM permissions; this is expected and does not affect instance termination.

---

### Step 4: Locate Target Instance

**Objective:** Identify and confirm the `xfusion-ec2` instance before taking any destructive action.

**Actions:**
1. In the left navigation panel, click **Instances → Instances**
2. The instance list will display with a default filter of `Instance state = running`
3. Locate the instance named **`xfusion-ec2`**
4. Click on the instance row to expand the **Details** pane and verify:

**Verification Checklist (Pre-Termination):**

| Attribute | Expected Value | Confirmed |
|-----------|----------------|-----------|
| Name | `xfusion-ec2` | ✅ |
| Instance ID | `i-0d87b86ea4f037cec` | ✅ |
| Instance State | `Running` | ✅ |
| Instance Type | `t2.micro` | ✅ |
| Availability Zone | `us-east-1a` | ✅ |
| Status Checks | `2/2 checks passed` | ✅ |
| Region | `us-east-1` | ✅ |

**Screenshot: Instance List:**

![Step 4: Instance Located](./screenshots/step4_instance_located.png)

> The instance is confirmed running with all status checks passing. The Instance ID in the Details pane matches the expected value, providing double confirmation before proceeding.

---

### Step 5: Terminate the Instance

**Objective:** Permanently terminate the `xfusion-ec2` instance.

**Actions:**
1. Ensure the checkbox next to `xfusion-ec2` is **selected** (highlighted in blue)
2. Click the **Instance state** dropdown button in the top-right action bar
3. Select **Terminate (delete) instance** from the dropdown menu
4. A confirmation dialog will appear; review the instance details displayed
5. Click **Terminate** to confirm and initiate the termination

**What Happens Next:**
- AWS immediately transitions the instance state to `Shutting-down`
- The hypervisor initiates a graceful shutdown sequence
- EBS root volumes with the `DeleteOnTermination` flag (default: `true`) will be automatically deleted
- The instance will reach `Terminated` state within 1-5 minutes

**Screenshot: Instance State Dropdown (Terminate Selected):**

![Step 5a: Terminate Option](./screenshots/step5a_terminate_dropdown.png)

**Screenshot: Termination Initiated (Shutting-down state):**

![Step 5b: Shutting Down](./screenshots/step5b_shutting_down.png)

> The instance state immediately transitions to `Shutting-down` following confirmation. The success banner at the top of the console confirms: *"Successfully initiated termination (deletion) of i-0d87b86ea4f037cec."*

---

### Step 6: Verify Termination State

**Objective:** Confirm the instance has fully reached the `Terminated` state, completing the cleanup operation.

**Actions:**
1. Wait 1-5 minutes for the termination process to complete
2. The instance list will refresh automatically, or click the **refresh** (🔄) icon
3. Observe the instance state transition from `Shutting-down` → `Terminated`
4. Click on the instance row and confirm the Details pane reflects:
   - **Instance state:** `Terminated`
   - **Public IPv4 address:** cleared (empty)
   - **Public DNS:** cleared (empty)
   - **Private IPv4:** cleared (empty)

**Screenshot: Instance Terminated:**

![Step 6: Terminated State](./screenshots/step6_terminated.png)

> The instance state is confirmed as `Terminated`. All network attributes (Public IPv4, Private IPv4, Public DNS) have been cleared, which is the expected behavior for a fully terminated instance. The success banner remains visible, confirming the operation completed successfully.

---

## 6. Final Verification Summary

| Parameter | Value |
|-----------|-------|
| **Instance Name** | `xfusion-ec2` |
| **Instance ID** | `i-0d87b86ea4f037cec` |
| **AWS Region** | `us-east-1` (US East (N. Virginia)) |
| **Availability Zone** | `us-east-1a` |
| **Pre-Operation State** | `Running` |
| **Post-Operation State** | `Terminated` ✅ |
| **Termination Confirmed Via** | AWS Console, Instance Details Pane |
| **Task Status** | **SUCCESSFUL** ✅ |

---

## 7. Troubleshooting & Edge Cases

**Instance not visible in the list:**
- Confirm the region is set to `us-east-1`
- Remove active filters (e.g., clear "Instance state = running" to show all states)
- The instance may have already been terminated in a prior operation

**"Terminate" option is grayed out:**
- Termination protection is enabled on the instance
- Navigate to **Actions → Instance Settings → Change Termination Protection** → Disable
- Retry termination after disabling

**Instance stuck in `Shutting-down` for more than 10 minutes:**
- This is rare but can occur with OS-level shutdown hooks
- AWS automatically forces termination after an extended period
- If urgent, use the AWS CLI: `aws ec2 terminate-instances --instance-ids i-0d87b86ea4f037cec --region us-east-1`

**IAM `AccessDenied` error on termination:**
- The operator role lacks `ec2:TerminateInstances` permission
- Escalate to an IAM administrator to grant the required permission or assume a higher-privilege role

**EBS volumes not deleted after termination:**
- Volumes with `DeleteOnTermination = false` persist after instance termination
- Check **Elastic Block Store → Volumes** for any unattached volumes and delete manually if no longer needed

---

## 8. Lessons Learned & Best Practices

**Always double-check Instance ID before terminating.** Instance names are mutable tags, but the Instance ID is the unique, immutable identifier. Cross-reference both before confirming any destructive action.

**Use resource tags for lifecycle management.** Tag instances with `Environment`, `Owner`, `ExpiryDate`, and `CostCenter` to simplify future audits and automated cleanup pipelines (e.g., AWS Config rules or Lambda-based schedulers).

**Consider Stop before Terminate for non-urgent decommissions.** Stopping an instance first allows time for final data extraction or AMI creation before permanent deletion.

**Snapshot EBS volumes with unknown data.** If the purpose of attached volumes is unclear, create a snapshot before termination. Snapshots are cheap; data recovery is not.

**Prefer IaC for resource lifecycle.** Resources provisioned via Terraform, CloudFormation, or CDK should be decommissioned through the same tooling (e.g., `terraform destroy`) to maintain state consistency and avoid drift.

**Document in change management systems.** Always log infrastructure deletions in a ticketing system (JIRA, ServiceNow) with the Instance ID, operator, timestamp, and justification for full auditability.

---

## 9. Tags & Metadata

**Operational Tags:** `aws` · `ec2` · `cloud-cleanup` · `infrastructure` · `devops`  
**Service:** Amazon EC2  
**Action Type:** Destructive / Decommission  
**Environment:** Production (Lab Account)  
**Executed By:** `kk_labs_us` (IAM User)  
**Account ID:** `kaml-prod-265 (6926-998-...)`  
**Execution Date:** 2026  
**Documentation Standard:** FAANG-Level Engineering Runbook













# AWS EC2 Instance Cleanup (us-east-1)

## PROJECT OVERVIEW
- During infrastructure migration, obsolete AWS resources were identified.
- An EC2 instance named "xfusion-ec2" is no longer required and must be deleted.
- The task ensures proper cleanup and confirms the instance reaches
the TERMINATED state.

## OBJECTIVES
- Identify EC2 instance named xfusion-ec2
- Ensure correct AWS region (us-east-1)
- Terminate the instance
- Verify instance is fully terminated

## HIGH-LEVEL WORKFLOW

- LOGIN to AWS Console
- SET region to us-east-1
- OPEN EC2 service
- SEARCH for instance named xfusion-ec2
- IF instance exists:
  -  TERMINATE instance
  -  WAIT for termination
- VERIFY instance state == terminated
- END task

## IMPLEMENTATION STEPS

## STEP 1: LOGIN TO AWS CONSOLE
- ACTION:
  -  OPEN browser
  -  NAVIGATE to AWS Console URL
  -  LOGIN using provided credentials

SCREENSHOT:
<img width="1819" height="942" alt="image" src="https://github.com/user-attachments/assets/302cb83e-2856-49fd-93f8-c423c4c1384b" />

## STEP 2: SET AWS REGION
- ACTION:
  -  SELECT region dropdown
  -  CHOOSE `us-east-1`

SCREENSHOT:
<img width="1818" height="945" alt="image" src="https://github.com/user-attachments/assets/ed4b9efc-85bd-47cc-9542-62b65d24f501" />

## STEP 3: OPEN EC2 DASHBOARD
- ACTION:
  -  NAVIGATE to Services
  -  SELECT EC2

SCREENSHOT:
<img width="1839" height="889" alt="image" src="https://github.com/user-attachments/assets/df932739-dbd6-42e2-99a7-2acce0c7841c" />

## STEP 4: LOCATE TARGET INSTANCE
- ACTION:
  -  OPEN Instances
  -  SEARCH for instance name "xfusion-ec2"

- EXPECTED RESULT:
  -  Instance visible in instance list

SCREENSHOT:
<img width="1840" height="893" alt="image" src="https://github.com/user-attachments/assets/8b9886c6-3473-40a2-be02-9db11770c87e" />

## STEP 5: TERMINATE INSTANCE
- ACTION:
  -  SELECT xfusion-ec2
  -  CLICK Instance State
  -  SELECT Terminate Instance
  -  CONFIRM termination

SCREENSHOT:
<img width="1802" height="892" alt="image" src="https://github.com/user-attachments/assets/51017336-e867-4939-b36a-deaf5995b441" />
<img width="1805" height="892" alt="image" src="https://github.com/user-attachments/assets/543dedde-d4b9-4136-ba33-84c4aee0c6d4" />

## STEP 6: VERIFY TERMINATION
- ACTION:
  -  WAIT until instance state changes
  -  CONFIRM state == terminated

SCREENSHOT:
<img width="1844" height="897" alt="image" src="https://github.com/user-attachments/assets/7ca6003f-9dfd-481d-9779-6a114a7ce22b" />

## FINAL VERIFICATION

- INSTANCE NAME:
`xfusion-ec2`

REGION:
`us-east-1`

FINAL STATE:
`terminated`

TASK STATUS:
`SUCCESSFUL`

## TAGS

`aws`
`ec2`
`cloud-cleanup`
`infrastructure`
`devops`













