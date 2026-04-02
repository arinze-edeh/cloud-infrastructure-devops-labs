# Provisioning EC2 Amazon Machine Images for Repeatable Infrastructure Deployments

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Objectives](#objectives)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Authenticate to AWS Console and Confirm Region](#step-1-authenticate-to-aws-console-and-confirm-region)
  - [Step 2: Navigate to EC2 and Locate Target Instance](#step-2-navigate-to-ec2-and-locate-target-instance)
  - [Step 3: Initiate AMI Creation from the Running Instance](#step-3-initiate-ami-creation-from-the-running-instance)
  - [Step 4: Configure AMI Metadata and Submit Creation Request](#step-4-configure-ami-metadata-and-submit-creation-request)
  - [Step 5: Monitor AMI Creation Progress](#step-5-monitor-ami-creation-progress)
  - [Step 6: Validate AMI Availability and Readiness](#step-6-validate-ami-availability-and-readiness)
- [Validation Summary](#validation-summary)
- [Operational Considerations](#operational-considerations)
- [Risks and Edge Cases](#risks-and-edge-cases)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Final Outcome](#final-outcome)
- [Tags](#tags)

---

## Overview

This document describes the end-to-end process for creating an Amazon Machine Image (AMI) from an existing EC2 instance in AWS. The procedure was executed as part of the Nautilus DevOps team's incremental infrastructure migration to AWS, establishing a reliable, repeatable deployment artifact from a running instance configuration.

AMIs serve as the foundational image layer for EC2 deployments, enabling consistent instance provisioning, disaster recovery, and horizontal scaling without manual reconfiguration.

---

## Problem Statement

The Nautilus DevOps team required a validated, reusable system image derived from a known-good EC2 instance. Manually replicating instance configurations at scale introduces configuration drift, human error, and operational overhead. The solution was to capture the current state of the `nautilus-ec2` instance as an AMI, enabling on-demand reproduction of that exact environment for future deployments, environment cloning, and recovery scenarios.

---

## Architecture Context

- **Target Instance:** `nautilus-ec2`
- **AWS Region:** `us-east-1` (N. Virginia)
- **AMI Name:** `nautilus-ec2-ami`
- **Scope:** Single root volume snapshot-backed AMI
- **Ownership:** Account-private (not shared publicly)

When an AMI is created from a running instance, AWS performs a point-in-time capture of all attached EBS volumes via snapshots. The resulting AMI includes the root device mapping and snapshot references required to launch new instances in an identical state.

---

## Prerequisites

- Active AWS account with IAM permissions including `ec2:CreateImage`, `ec2:DescribeInstances`, and `ec2:DescribeImages`
- An existing, running EC2 instance (`nautilus-ec2`) in `us-east-1`
- Console access via a supported browser
- Familiarity with AWS EC2 instance lifecycle management

---

## Objectives

- Identify the existing EC2 instance `nautilus-ec2` in the `us-east-1` region
- Create an AMI named `nautilus-ec2-ami` from that instance
- Confirm the AMI reaches the `available` state, indicating successful snapshot completion
- Validate the AMI is account-owned and ready for use in future EC2 launch operations

---

## Implementation Steps

### Step 1: Authenticate to AWS Console and Confirm Region

Open the AWS Management Console and authenticate using the provisioned IAM credentials. Before proceeding, confirm the active region is set to **US East (N. Virginia) / us-east-1** via the region selector in the top-right navigation bar. All subsequent operations must be performed within this region to ensure the AMI and source instance are co-located.

> **Operational Note:** Performing cross-region AMI operations without explicit intent can result in the AMI being created in an unintended region, making it invisible when searching from the source region. Always verify region context before executing infrastructure changes.

**Screenshot: AWS Console authenticated with us-east-1 region confirmed**

![Step 1 - AWS Console Login and Region Selection](https://github.com/user-attachments/assets/30502e7a-8fae-4566-93dc-bfeddb7d7a1c)

---

### Step 2: Navigate to EC2 and Locate Target Instance

From the AWS Console home, navigate to **EC2** via the Services menu or search bar. In the left navigation panel, select **Instances** under the Instances section. Locate the instance named **nautilus-ec2** in the instance list.

Verify the following before proceeding:
- Instance state is **running** or **stopped** (AMI creation is supported in both states)
- Instance is in the correct region (`us-east-1`)
- The correct instance is selected to prevent accidental imaging of the wrong resource

> **Operational Note:** Creating an AMI from a running instance does not require a reboot by default. However, AWS cannot guarantee filesystem consistency for in-memory writes unless a reboot is performed or the instance is stopped prior to imaging. For critical workloads, evaluate whether a no-reboot AMI is acceptable or if instance shutdown is required for a crash-consistent image.

**Screenshot: EC2 Instances view with nautilus-ec2 identified**

![Step 2 - EC2 Instance List Showing nautilus-ec2](https://github.com/user-attachments/assets/cf69143b-126a-4e75-998d-2171c3803bab)

---

### Step 3: Initiate AMI Creation from the Running Instance

With **nautilus-ec2** selected, trigger the AMI creation workflow via the console:

1. Click the **Actions** dropdown in the top-right of the instance list
2. Hover over **Image and templates**
3. Select **Create image**

This opens the AMI creation configuration panel, where image metadata and volume settings can be defined.

> **Operational Note:** The "No reboot" option in the creation form allows the instance to remain online during imaging. When left unchecked (default behavior), AWS will reboot the instance to flush disk caches and achieve a consistent filesystem state before snapshotting. For production environments, plan this reboot during a maintenance window.

**Screenshot: Actions menu with Image and templates and Create image selected**

![Step 3 - Actions Menu for AMI Creation](https://github.com/user-attachments/assets/1e8e3179-629c-4970-9e98-c6b755bf1a7e)

---

### Step 4: Configure AMI Metadata and Submit Creation Request

In the **Create image** panel, configure the following:

- **Image name:** `nautilus-ec2-ami`
- **Image description:** *(Optional but recommended for production use)* Add a description summarizing the instance purpose and capture date
- **No reboot:** Leave unchecked for filesystem consistency (default)
- **Instance volumes:** Verify that the root volume is included with its current configuration; additional EBS volumes attached to the instance will also appear here

After confirming all settings, click **Create image** to submit the request.

AWS will immediately return an AMI ID confirming the creation task has been queued. The actual snapshot process runs asynchronously and may take several minutes depending on volume size.

> **Best Practice:** Adopt a consistent AMI naming convention in production environments, such as `{service}-{environment}-{date}` (for example, `nautilus-ec2-prod-2024-07-15`). This enables fast identification, auditing, and lifecycle management at scale.

**Screenshot: AMI creation form with nautilus-ec2-ami as the image name**

![Step 4 - AMI Configuration Form](https://github.com/user-attachments/assets/86838675-4cf1-4eb6-b4a5-9e1ad27cf5b4)

---

### Step 5: Monitor AMI Creation Progress

Navigate to the **AMIs** section from the EC2 left navigation panel (under **Images**). Locate **nautilus-ec2-ami** in the AMI list.

Monitor the **State** column. The AMI will initially appear as `pending` while AWS performs the following background operations:
- Initiates EBS snapshot(s) of all attached volumes
- Registers the AMI with the associated snapshot mappings
- Transitions the AMI state to `available` upon completion

Refresh the page periodically until the status changes to **available**.

> **Operational Note:** AMI creation duration scales with the total size of attached volumes. A root volume of several GBs typically completes within 2 to 10 minutes. Larger data volumes or instances with multiple attached EBS volumes may take significantly longer. For automation pipelines, use `aws ec2 wait image-available --image-ids <ami-id>` to programmatically block on completion.

**Screenshot: AMI list showing nautilus-ec2-ami in pending state transitioning to available**

![Step 5 - AMI Status Monitoring](https://github.com/user-attachments/assets/0b007ee7-b952-4cda-8860-d366cf4a3861)

---

### Step 6: Validate AMI Availability and Readiness

Once the AMI state shows **available**, perform the following verification checks:

- **State confirmation:** AMI status column reads `available`
- **Ownership:** AMI is listed under **Owned by me**, confirming it is account-private
- **AMI ID:** Note and record the AMI ID for downstream reference in launch templates, IaC configurations, or runbooks
- **Snapshot association:** Confirm the root snapshot is listed under **Storage** in the AMI details panel

The AMI is now ready for use in EC2 launch operations, Auto Scaling group configurations, or infrastructure-as-code tooling (Terraform, CloudFormation).

**Screenshot: AMI detail view confirming available state and account ownership**

![Step 6 - AMI Available State Confirmed](https://github.com/user-attachments/assets/8f892b1d-576d-4f36-b16a-1e6400649aaf)

---

## Validation Summary

| Checkpoint | Expected Result | Status |
|---|---|---|
| Region confirmed as us-east-1 | Console region selector shows N. Virginia | Verified |
| nautilus-ec2 instance located | Instance visible in EC2 Instances list | Verified |
| AMI creation initiated | AMI ID returned immediately after submission | Verified |
| AMI name set correctly | Name displays as `nautilus-ec2-ami` | Verified |
| AMI state transitions to available | State column reads `available` | Verified |
| AMI ownership confirmed | Listed under "Owned by me" | Verified |

---

## Operational Considerations

- **Snapshot costs:** Each AMI creation generates one or more EBS snapshots. These snapshots incur ongoing storage charges in AWS. Implement a lifecycle policy to deregister outdated AMIs and delete orphaned snapshots to control costs.
- **Cross-region replication:** If the AMI is needed in additional regions for multi-region deployments or disaster recovery, use the **Copy AMI** feature to replicate it across regions.
- **Launch permissions:** By default, AMIs are private to the owning AWS account. Explicitly configure launch permissions if the AMI needs to be shared with other accounts.
- **Encryption:** If the source instance uses encrypted EBS volumes, the resulting AMI snapshots will also be encrypted. Unencrypted volumes produce unencrypted snapshots. Enforce encryption at the account level using EBS default encryption policies for compliance-sensitive environments.

---

## Risks and Edge Cases

- **Running instance imaging:** AMIs created from running instances without a reboot may not guarantee application-level consistency. Databases or services with in-memory state may capture incomplete writes. Evaluate no-reboot AMIs only for stateless workloads.
- **Snapshot failure mid-creation:** If an underlying EBS snapshot fails during AMI creation, the AMI may remain in `pending` state indefinitely or transition to a `failed` state. Monitor the AMI status and associated snapshot states under **EC2 > Snapshots**.
- **IAM permission gaps:** Insufficient IAM permissions (missing `ec2:CreateImage`, `ec2:CreateSnapshot`) will silently fail or return an access denied error. Verify permissions before initiating the workflow in a new account or role context.
- **Volume size limits:** Extremely large EBS volumes can increase snapshot time significantly. For instances with multi-TB volumes, plan AMI creation during off-peak hours.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| AMI remains in `pending` state for extended period | Large volume size or snapshot bottleneck | Monitor snapshot progress under EC2 > Snapshots; allow additional time |
| AMI not visible in AMI list | Wrong region selected in console | Confirm region is set to us-east-1 |
| Access denied on Create image | IAM role missing `ec2:CreateImage` or `ec2:CreateSnapshot` | Attach the appropriate IAM policy to the role |
| AMI in `failed` state | Underlying snapshot failure | Deregister the AMI, delete the failed snapshot, and retry |
| Instance rebooted unexpectedly during imaging | "No reboot" option was left unchecked | Expected behavior; plan reboot during maintenance windows for production |

---

## Best Practices

- **Naming conventions:** Use structured, searchable AMI names that encode the service name, environment, and date (for example, `nautilus-ec2-prod-20240715`). This simplifies filtering and lifecycle management at scale.
- **Tagging:** Apply resource tags to AMIs at creation time (`Environment`, `Owner`, `Project`, `CreatedDate`). Tags enable cost allocation, access control, and automated lifecycle policies.
- **Lifecycle automation:** Use AWS Data Lifecycle Manager (DLM) or custom Lambda-based automation to periodically deregister old AMIs and purge associated snapshots based on retention policies.
- **Pre-imaging validation:** Before capturing an AMI, ensure the instance is in a known-good state. Run application health checks, clear temporary files, and confirm services are behaving as expected.
- **Documentation:** Record the AMI ID, creation date, source instance ID, and intended use case in team runbooks or infrastructure asset registers for auditability.

---

## Lessons Learned

- AMI creation is asynchronous. The console submission confirms task initiation, not completion. Always monitor AMI state to confirm the `available` transition before using the image in downstream processes.
- The "No reboot" option is not always safe for production workloads. For stateful services, stopping the instance before imaging is the most reliable path to a consistent snapshot.
- Untagged AMIs and orphaned snapshots accumulate quickly in active AWS accounts and generate avoidable costs. Establish tagging and lifecycle policies before scaling AMI creation workflows.
- In environments with strict IAM permission boundaries, verifying role permissions for `ec2:CreateImage` and `ec2:CreateSnapshot` before executing the workflow prevents unexpected failures during time-sensitive operations.

---

## Final Outcome

- AMI successfully created from the existing EC2 instance `nautilus-ec2`
- AMI correctly named `nautilus-ec2-ami` in the us-east-1 region
- AMI reached `available` state, confirming all underlying EBS snapshots completed successfully
- AMI is account-private and ready for use in EC2 launch operations, Auto Scaling configurations, and infrastructure-as-code pipelines

---

## Tags

`aws` `ec2` `ami` `ebs-snapshots` `cloud-migration` `devops` `infrastructure-as-code` `image-management` `repeatable-deployments` `nautilus-devops`


















# EC2 AMI Creation from Existing Instance

## 📌 Lab Overview
- The Nautilus DevOps team is migrating infrastructure to AWS using an
incremental approach. As part of this process, an Amazon Machine Image
(AMI) was created from an existing EC2 instance to enable repeatable
deployments, backups, and scaling operations.

---

## 🎯 Objectives
- Identify an existing EC2 instance
- Create an AMI from the instance
- Ensure AMI reaches `available` state
- Validate successful image creation

---

## 🧠 High-Level Logic

- LOGIN to AWS Console
- SELECT region us-east-1

- LOCATE EC2 instance "nautilus-ec2"

- INITIATE AMI creation:
  -  SET image name = "nautilus-ec2-ami"
  -  START image creation process

- WAIT until AMI status becomes "available"

- VERIFY AMI exists and is usable

## 🛠️ Implementation Steps

## Step 1: Login to AWS Console
- Open AWS Console URL

- Authenticate using provided credentials

- Ensure region is set to us-east-1

📸 screenshot:
<img width="1791" height="941" alt="image" src="https://github.com/user-attachments/assets/30502e7a-8fae-4566-93dc-bfeddb7d7a1c" />

## Step 2: Navigate to EC2 Service
- Open EC2 Dashboard

- Select Instances

- Locate instance named nautilus-ec2

📸 screenshot:
<img width="1785" height="937" alt="image" src="https://github.com/user-attachments/assets/cf69143b-126a-4e75-998d-2171c3803bab" />

## Step 3: Create AMI from EC2 Instance
- Select the instance nautilus-ec2

- Click Actions

- Choose Image and templates

- Click Create image

📸 screenshot:
<img width="1793" height="944" alt="image" src="https://github.com/user-attachments/assets/1e8e3179-629c-4970-9e98-c6b755bf1a7e" />


## Step 4: Configure AMI Details
- AMI Name: nautilus-ec2-ami

- Leave other settings as default

- Click Create image

📸 screenshot:
<img width="1821" height="943" alt="image" src="https://github.com/user-attachments/assets/86838675-4cf1-4eb6-b4a5-9e1ad27cf5b4" />

## Step 5: Monitor AMI Creation Status
- Navigate to AMIs

- Locate nautilus-ec2-ami

- Wait until status changes to available

📸 screenshot:
<img width="1856" height="940" alt="image" src="https://github.com/user-attachments/assets/0b007ee7-b952-4cda-8860-d366cf4a3861" />

## Step 6: Verify AMI Availability
- Confirm AMI state is available

- Ensure AMI is owned by current account

- Validate readiness for EC2 launches

📸 screenshot:
<img width="1861" height="945" alt="image" src="https://github.com/user-attachments/assets/8f892b1d-576d-4f36-b16a-1e6400649aaf" />

## ✅ Final Outcome
- AMI successfully created from existing EC2 instance

- AMI name set correctly as nautilus-ec2-ami

- Image reached available state

- AMI ready for reuse, scaling, and recovery operations

## 🏷️ Tags
`aws` `ec2` `ami` `snapshots` `cloud-migration` `devops` `infrastructure`
