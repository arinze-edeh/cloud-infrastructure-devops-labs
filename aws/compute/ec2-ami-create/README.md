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
  - [Step 3: Configure AMI Metadata and Submit Creation Request](#step-3-configure-ami-metadata-and-submit-creation-request)
  - [Step 4: Confirm AMI Creation Initiation](#step-4-confirm-ami-creation-initiation)
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

| Attribute | Value |
|---|---|
| **Target Instance** | `nautilus-ec2` |
| **Instance ID** | `i-0ae06eb1eee1a28d3` |
| **Instance Type** | `t2.micro` |
| **Availability Zone** | `us-east-1c` |
| **AWS Region** | `us-east-1` (N. Virginia) |
| **AMI Name** | `nautilus-ec2-ami` |
| **AMI ID** | `ami-038c9bf484956b9ad` |
| **Root Device** | `/dev/xvda` (8 GiB, EBS General Purpose SSD gp3) |
| **Architecture** | `x86_64` |
| **Virtualization Type** | `hvm` |
| **Boot Mode** | `uefi-preferred` |
| **Visibility** | Private (account-owned) |
| **Owner Account ID** | `519689500096` |
| **Creation Date** | `2026-02-14T00:43:43.000Z` |

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

The region dropdown confirms `N. Virginia` highlighted as the active selection (`us-east-1`), which is the correct target region for this operation.

> **Operational Note:** Performing cross-region AMI operations without explicit intent can result in the AMI being created in an unintended region, making it invisible when searching from the source region. Always verify region context before executing infrastructure changes.

**Screenshot: AWS Console authenticated with region selector open confirming us-east-1 (N. Virginia) as the active region**

![Step 1 - AWS Console Login and Region Selection](images/step-01-aws-console-region-us-east-1.png)

---

### Step 2: Navigate to EC2 and Locate Target Instance

From the AWS Console home, navigate to **EC2** via the Services menu or search bar. In the left navigation panel, select **Instances** under the Instances section. Locate the instance named **nautilus-ec2** in the instance list.

The instance list confirms the following attributes for `nautilus-ec2`:

- **Instance ID:** `i-0ae06eb1eee1a28d3`
- **Instance State:** `Running`
- **Instance Type:** `t2.micro`
- **Status Check:** `2/2 checks passed`
- **Availability Zone:** `us-east-1c`
- **Public IPv4:** `98.93.36.255`

Verify all of the above before proceeding. A running instance in a healthy state (2/2 checks passed) is the optimal source for AMI capture.

> **Operational Note:** Creating an AMI from a running instance does not require a reboot by default. However, AWS cannot guarantee filesystem consistency for in-memory writes unless a reboot is performed or the instance is stopped prior to imaging. For critical workloads, evaluate whether a no-reboot AMI is acceptable or if instance shutdown is required for a crash-consistent image.

**Screenshot: EC2 Instances view showing nautilus-ec2 in Running state with all health checks passing**

![Step 2 - EC2 Instance List Showing nautilus-ec2](images/step-02-ec2-instance-nautilus-ec2-running.png)

---

### Step 3: Configure AMI Metadata and Submit Creation Request

With **nautilus-ec2** selected, navigate to **Actions > Image and templates > Create image**. This opens the **Create image** configuration panel. Configure the following settings:

- **Instance ID:** `i-0ae06eb1eee1a28d3` (nautilus-ec2) -- pre-populated and confirmed
- **Image name:** `nautilus-ec2-ami`
- **Image description:** Optional; recommended for production documentation
- **Reboot instance:** Checked (default) -- AWS will reboot the instance before snapshotting to ensure data consistency
- **Instance volumes:** One EBS volume visible (`/dev/xvda`, 8 GiB, EBS General Purpose SSD gp3, 3000 IOPS, Delete on termination enabled)
- **Tags:** "Tag image and snapshots together" selected -- ensures consistent tagging across the AMI and its backing snapshots

After confirming all settings, scroll down and click **Create image** to submit the request.

> **Best Practice:** The "Reboot instance" checkbox is enabled by default and is the recommended setting for production workloads. It ensures AWS can flush in-memory buffers and capture a filesystem-consistent snapshot. Only disable this option for stateless workloads where a brief reboot is not acceptable.

> **Best Practice:** Adopt a consistent AMI naming convention in production environments, such as `{service}-{environment}-{date}` (for example, `nautilus-ec2-prod-20260214`). This enables fast identification, auditing, and lifecycle management at scale.

**Screenshot: Create image form populated with nautilus-ec2-ami as the image name, Reboot instance enabled, and root EBS volume confirmed at 8 GiB gp3**

![Step 3 - AMI Configuration Form](images/step-03-create-image-form-nautilus-ec2-ami.png)

---

### Step 4: Confirm AMI Creation Initiation

After clicking **Create image**, AWS immediately registers the AMI and returns an AMI ID. The console displays a confirmation banner at the top of the Instances view:

> *"Currently creating AMI `ami-038c9bf484956b9ad` from instance `i-0ae06eb1eee1a28d3`. Check that the AMI status is 'Available' before deleting the instance or carrying out other actions related to this AMI."*

This banner confirms the AMI creation task has been successfully queued. The underlying EBS snapshot process is now running asynchronously in the background. The source instance `nautilus-ec2` continues running during this process.

> **Operational Note:** Do not terminate or stop the source instance while the AMI creation is in progress. Interrupting the instance during snapshot capture can result in a failed or corrupt AMI. Wait until the AMI reaches `available` state before performing any disruptive operations on the source instance.

**Screenshot: EC2 Instances view showing the green confirmation banner with AMI ID ami-038c9bf484956b9ad and source instance details**

![Step 4 - AMI Creation Confirmation Banner](images/step-04-ami-creation-confirmation-banner.png)

---

### Step 5: Monitor AMI Creation Progress

Navigate to **EC2 > Images > AMIs** from the left navigation panel. The AMI list view, filtered to **Owned by me**, displays `nautilus-ec2-ami` with the following confirmed attributes:

- **AMI name:** `nautilus-ec2-ami`
- **AMI ID:** `ami-038c9bf484956b9ad`
- **Source:** `519689500096/nautilus-ec2-ami`
- **Owner:** `519689500096`
- **Visibility:** `Private`
- **Status:** `Available`
- **Creation date:** `2026/02/14 01:43`

The AMI has successfully transitioned from `pending` to `available`, confirming all underlying EBS snapshots completed without error.

> **Operational Note:** AMI creation duration scales with the total size of attached volumes. An 8 GiB root volume typically completes within 2 to 5 minutes under normal conditions. For automation pipelines, use the AWS CLI command `aws ec2 wait image-available --image-ids ami-038c9bf484956b9ad` to programmatically block downstream steps until the AMI is ready.

**Screenshot: AMIs list view filtered to Owned by me, showing nautilus-ec2-ami with Available status, Private visibility, and creation timestamp**

![Step 5 - AMI Available in AMI List](images/step-05-ami-list-available-status.png)

---

### Step 6: Validate AMI Availability and Readiness

Select `nautilus-ec2-ami` in the AMI list to expand the detail panel. Confirm the following attributes against expected values:

| Attribute | Observed Value | Expected |
|---|---|---|
| AMI name | `nautilus-ec2-ami` | `nautilus-ec2-ami` |
| AMI ID | `ami-038c9bf484956b9ad` | Matches banner confirmation |
| Status | `Available` | `Available` |
| Owner account ID | `519689500096` | Current account |
| Architecture | `x86_64` | Matches source instance |
| Virtualization type | `hvm` | Standard for t2.micro |
| Boot mode | `uefi-preferred` | Inherited from source |
| Root device name | `/dev/xvda` | EBS-backed root volume |
| Block devices | `/dev/xvda=snap-03865c4a553827e67:8:true:gp3` | 8 GiB snapshot confirmed |
| Source AMI region | `us-east-1` | Correct region |
| Deregistration protection | `Disabled` | Default; enable for critical AMIs |

All validation checks pass. The AMI is confirmed as account-private, architecture-compatible, and backed by a completed EBS snapshot. It is ready for use in EC2 launch operations, Auto Scaling group configurations, Launch Templates, and infrastructure-as-code tooling (Terraform, CloudFormation).

**Screenshot: AMI detail panel for ami-038c9bf484956b9ad showing all metadata including Available status, block device mapping, architecture, and owner account confirmation**

![Step 6 - AMI Detail Panel with Full Metadata Validation](images/step-06-ami-detail-panel-full-validation.png)

---

## Validation Summary

| Checkpoint | Expected Result | Observed Result | Status |
|---|---|---|---|
| Region confirmed as us-east-1 | N. Virginia active in region selector | N. Virginia highlighted in dropdown | Passed |
| nautilus-ec2 instance located | Instance visible in Running state | Running, 2/2 checks passed | Passed |
| AMI creation initiated | Green confirmation banner with AMI ID | Banner shows `ami-038c9bf484956b9ad` | Passed |
| AMI name set correctly | `nautilus-ec2-ami` | `nautilus-ec2-ami` confirmed | Passed |
| AMI transitions to available | Status column reads `Available` | `Available` confirmed in AMI list | Passed |
| AMI ownership confirmed | Listed under "Owned by me" | Owner: `519689500096`, Visibility: Private | Passed |
| Block device snapshot confirmed | Root volume snapshot attached | `/dev/xvda=snap-03865c4a553827e67:8:true:gp3` | Passed |

---

## Operational Considerations

- **Snapshot costs:** Each AMI creation generates one or more EBS snapshots. These snapshots incur ongoing storage charges in AWS. Implement a lifecycle policy to deregister outdated AMIs and delete orphaned snapshots to control costs.
- **Cross-region replication:** If the AMI is needed in additional regions for multi-region deployments or disaster recovery, use the **Copy AMI** feature to replicate it across regions. The copied AMI will receive a new AMI ID in the target region.
- **Launch permissions:** By default, AMIs are private to the owning AWS account. Explicitly configure launch permissions if the AMI needs to be shared with other accounts or made public.
- **Encryption:** If the source instance uses encrypted EBS volumes, the resulting AMI snapshots will also be encrypted. Unencrypted volumes produce unencrypted snapshots. Enforce encryption at the account level using EBS default encryption policies for compliance-sensitive environments.
- **Deregistration protection:** The `nautilus-ec2-ami` was created with deregistration protection disabled. For production AMIs used in Auto Scaling or golden image pipelines, consider enabling deregistration protection to prevent accidental deletion.

---

## Risks and Edge Cases

- **Running instance imaging:** AMIs created from running instances without a reboot may not guarantee application-level consistency. Databases or services with in-memory state may capture incomplete writes. Evaluate no-reboot AMIs only for stateless workloads.
- **Snapshot failure mid-creation:** If an underlying EBS snapshot fails during AMI creation, the AMI may remain in `pending` state indefinitely or transition to a `failed` state. Monitor the AMI status and associated snapshot states under **EC2 > Snapshots**.
- **IAM permission gaps:** Insufficient IAM permissions (missing `ec2:CreateImage`, `ec2:CreateSnapshot`) will silently fail or return an access denied error. Verify permissions before initiating the workflow in a new account or role context.
- **Volume size limits:** Extremely large EBS volumes can increase snapshot time significantly. For instances with multi-TB volumes, plan AMI creation during off-peak hours.
- **Source instance termination during imaging:** Terminating the source instance while a snapshot is in progress can result in a failed or incomplete AMI. The console banner displayed in Step 4 explicitly warns against this. Always wait for `available` state before performing disruptive operations on the source.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| AMI remains in `pending` state for extended period | Large volume size or AWS snapshot throttling | Monitor snapshot progress under EC2 > Snapshots; allow additional time or retry during off-peak hours |
| AMI not visible in AMI list | Wrong region selected in console | Confirm region is set to us-east-1; use the AMI ID to search across all regions if needed |
| Access denied on Create image | IAM role missing `ec2:CreateImage` or `ec2:CreateSnapshot` | Attach the `AmazonEC2FullAccess` policy or a scoped custom policy to the role |
| AMI in `failed` state | Underlying EBS snapshot failure | Deregister the failed AMI, delete the associated failed snapshot, and retry |
| Instance rebooted unexpectedly during imaging | "Reboot instance" checkbox was left enabled (default) | Expected behavior when reboot is enabled; plan around maintenance windows for production instances |
| AMI visible but not launchable | AMI snapshot still completing or permissions issue | Confirm block device mapping shows a completed snapshot ID, not `snap-ffffffff` |

---

## Best Practices

- **Naming conventions:** Use structured, searchable AMI names that encode the service name, environment, and date (for example, `nautilus-ec2-prod-20260214`). This simplifies filtering and lifecycle management at scale.
- **Tagging:** Apply resource tags to AMIs at creation time (`Environment`, `Owner`, `Project`, `CreatedDate`). Tags enable cost allocation, access control, and automated lifecycle policies. Use the "Tag image and snapshots together" option to ensure consistent tagging across the AMI and its backing snapshots.
- **Lifecycle automation:** Use AWS Data Lifecycle Manager (DLM) or custom Lambda-based automation to periodically deregister old AMIs and purge associated snapshots based on retention policies.
- **Pre-imaging validation:** Before capturing an AMI, ensure the instance is in a known-good state. Run application health checks, clear temporary files, and confirm services are behaving as expected.
- **Deregistration protection:** Enable deregistration protection on critical golden AMIs used in production Auto Scaling groups or Launch Templates to prevent accidental deletion during housekeeping operations.
- **Documentation:** Record the AMI ID, creation date, source instance ID, and intended use case in team runbooks or infrastructure asset registers for auditability.

---

## Lessons Learned

- AMI creation is asynchronous. The console submission and confirmation banner confirm task initiation, not completion. Always navigate to the AMIs list and confirm the `available` state before referencing the AMI in downstream processes.
- The "Reboot instance" checkbox is enabled by default and is the correct setting for most workloads. Disabling it (no-reboot) is only appropriate for stateless or read-only workloads where filesystem consistency is not a concern.
- The "Tag image and snapshots together" option is valuable for cost visibility. Snapshots created without tags are difficult to associate with their parent AMI when auditing costs or cleaning up orphaned resources.
- The console banner in Step 4 explicitly warns against deleting the source instance before the AMI reaches `available` state. This is an easy step to overlook in automated pipelines. Build in a wait condition (`aws ec2 wait image-available`) before any downstream instance operations.
- Untagged AMIs and orphaned snapshots accumulate quickly in active AWS accounts and generate avoidable costs. Establish tagging and lifecycle policies before scaling AMI creation workflows.

---

## Final Outcome

- AMI successfully created from the existing EC2 instance `nautilus-ec2` (`i-0ae06eb1eee1a28d3`)
- AMI correctly named `nautilus-ec2-ami` and registered as `ami-038c9bf484956b9ad` in `us-east-1`
- AMI reached `available` state, confirming the root EBS snapshot (`snap-03865c4a553827e67`, 8 GiB, gp3) completed successfully
- AMI is account-private (Visibility: Private, Owner: `519689500096`) and ready for use in EC2 launch operations, Auto Scaling configurations, and infrastructure-as-code pipelines

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
