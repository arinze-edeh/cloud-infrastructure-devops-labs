# Attaching an Existing Elastic Network Interface (ENI) to a Running EC2 Instance via AWS CLI

> **Environment:** AWS us-east-1 | **Tooling:** AWS CLI | **Scope:** Network Interface Management

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Architecture Context](#architecture-context)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Authenticate to AWS](#step-1-authenticate-to-aws)
  - [Step 2: Verify AWS Region Configuration](#step-2-verify-aws-region-configuration)
  - [Step 3: Retrieve the EC2 Instance ID](#step-3-retrieve-the-ec2-instance-id)
  - [Step 4: Retrieve the ENI ID](#step-4-retrieve-the-eni-id)
  - [Step 5: Attach the ENI to the EC2 Instance](#step-5-attach-the-eni-to-the-ec2-instance)
  - [Step 6: Verify ENI Attachment Status](#step-6-verify-eni-attachment-status)
- [Validation and Final Outcome](#validation-and-final-outcome)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting](#troubleshooting)
- [Tags](#tags)

---

## Overview

This document provides a step-by-step operational guide for attaching a pre-existing **Elastic Network Interface (ENI)** to a running **Amazon EC2 instance** using the AWS CLI. The procedure follows a structured authenticate-identify-attach-verify lifecycle and is intended for use in production environments, onboarding workflows, and infrastructure handoff documentation.

---

## Problem Statement

During an incremental AWS infrastructure migration, the Nautilus DevOps team required an existing ENI (tagged `nautilus-eni`) to be attached to a currently running EC2 instance (tagged `nautilus-ec2`) in the `us-east-1` region. The ENI was pre-provisioned separately and needed to be associated with the instance at **device index 1**, preserving the existing primary network interface at index 0.

Failure to correctly attach the ENI would result in the instance missing its secondary network path, potentially disrupting planned traffic routing, dual-homing configurations, or security group isolation requirements during the migration.

---

## Prerequisites

- AWS CLI installed and configured with valid credentials
- IAM permissions for `ec2:DescribeInstances`, `ec2:DescribeNetworkInterfaces`, and `ec2:AttachNetworkInterface`
- The target EC2 instance (`nautilus-ec2`) is in a **running** state
- The ENI (`nautilus-eni`) is in an **available** state (not currently attached)
- The ENI and EC2 instance must reside in the **same Availability Zone**

---

## Architecture Context

An Elastic Network Interface (ENI) is a logical networking component that represents a virtual network card in a VPC. Attaching a secondary ENI to a running EC2 instance allows the instance to:

- Serve traffic on multiple subnets or IP addresses
- Maintain separate security group policies per interface
- Support high-availability failover patterns (ENI can be detached and reattached across instances)
- Facilitate network traffic isolation between application tiers

In this context, the ENI is attached as a secondary interface (`--device-index 1`), leaving the primary interface (`eth0`) untouched.

---

## Implementation Steps

### Step 1: Authenticate to AWS

Before executing any AWS CLI commands, confirm that valid session credentials are in place. Run the `showcreds` command to display the active credential context, including the console URL, IAM username, and session expiry time.

```bash
showcreds
```

**Expected output:** A table displaying the AWS Console URL, IAM username (`kk_labs_user_456753`), and session end time.

> **Note:** Ensure the session has not expired before proceeding. Sessions with imminent expiry risk partial command execution, which can leave resources in an inconsistent state.

**Screenshot: Active AWS credential session details**

![Step 1 - AWS Credentials](https://github.com/user-attachments/assets/77d11d15-16ae-47bf-bc73-49410931eb3f)

---

### Step 2: Verify AWS Region Configuration

Confirm that the AWS CLI is targeting the correct region (`us-east-1`) before issuing any resource queries. A misconfigured region is one of the most common causes of empty query results or unintended resource modifications.

```bash
aws configure get region
```

**Expected output:**
```
us-east-1
```

**Screenshot: Confirmed region configuration output**

![Step 2 - Region Verification](https://github.com/user-attachments/assets/0f796f77-aa0f-46ab-b7bc-cd14170b0bd7)

---

### Step 3: Retrieve the EC2 Instance ID

Query the EC2 service using a tag-based filter to identify the target instance. Using tag filters rather than hardcoded instance IDs makes this workflow portable and repeatable across environments.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
  --query "Reservations[].Instances[].InstanceId" \
  --output text
```

**Expected output:**
```
i-0c433b269b876a504
```

> **Best Practice:** Always use tag-based lookups in scripted workflows to avoid dependency on static resource IDs, which change across environment refreshes or recreations.

**Screenshot: EC2 instance ID retrieved via tag filter**

![Step 3 - EC2 Instance ID](https://github.com/user-attachments/assets/c411cd87-4d10-4de1-abf3-3dca3d046baf)

---

### Step 4: Retrieve the ENI ID

Similarly, query the ENI using its tag name to obtain the network interface ID programmatically. Confirm the returned ID before proceeding to the attachment step.

```bash
aws ec2 describe-network-interfaces \
  --filters "Name=tag:Name,Values=nautilus-eni" \
  --query "NetworkInterfaces[].NetworkInterfaceId" \
  --output text
```

**Expected output:**
```
eni-0642d9bc26cae7164
```

> **Edge Case:** If this query returns no results, confirm the ENI exists in the correct region and that its `Name` tag value matches exactly (tags are case-sensitive).

**Screenshot: ENI ID retrieved via tag filter**

![Step 4 - ENI ID](https://github.com/user-attachments/assets/8dddd358-87d6-4f2e-b318-caed828c8f67)

---

### Step 5: Attach the ENI to the EC2 Instance

With both IDs confirmed, execute the attachment command using `device-index 1` to assign the ENI as the secondary network interface. The primary interface (`eth0`) occupies index 0 and must not be targeted.

```bash
aws ec2 attach-network-interface \
  --network-interface-id eni-0642d9bc26cae7164 \
  --instance-id i-0c433b269b876a504 \
  --device-index 1
```

**Expected output:**
```json
{
    "AttachmentId": "eni-attach-0d3a3b1ca04fcd45d",
    "NetworkCardIndex": 0
}
```

> **Important:** The `AttachmentId` returned (`eni-attach-0d3a3b1ca04fcd45d`) is the handle used to manage or detach this specific attachment. Record it if the attachment may need to be reversed later.

> **Risk:** Specifying an incorrect `--device-index` can conflict with existing interfaces or fail silently on instance types that do not support multiple network cards. Always verify the instance type's ENI limits before attaching additional interfaces.

**Screenshot: ENI attachment command execution and returned attachment ID**

![Step 5 - ENI Attachment](https://github.com/user-attachments/assets/69392dba-668e-462a-988d-288f1cda754d)

---

### Step 6: Verify ENI Attachment Status

After attachment, confirm that the ENI status has transitioned from `available` to `in-use`. This validates that the attachment was registered successfully by the EC2 control plane.

**Check ENI status:**

```bash
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-0642d9bc26cae7164 \
  --query "NetworkInterfaces[].Status" \
  --output table
```

**Expected output:**
```
|DescribeNetworkInterfaces|
+-------------------------+
|  in-use                 |
+-------------------------+
```

**Confirm instance health and both ENIs are registered:**

```bash
aws ec2 describe-instance-status --instance-ids i-0c433b269b876a504
```

**Expected output (key fields):**
```json
{
    "AvailabilityZone": "us-east-1a",
    "InstanceId": "i-0c433b269b876a504",
    "InstanceState": { "Code": 16, "Name": "running" },
    "InstanceStatus": { "Status": "ok" },
    "SystemStatus": { "Status": "ok" }
}
```

**Confirm the ENI is registered on the instance:**

```bash
aws ec2 describe-instances \
  --instance-ids i-0c433b269b876a504 \
  --query "Reservations[].Instances[].NetworkInterfaces[].NetworkInterfaceId"
```

**Expected output:**
```json
[
    "eni-0e02b491a8002ff0c",
    "eni-0642d9bc26cae7164"
]
```

Both the original primary ENI and the newly attached secondary ENI (`eni-0642d9bc26cae7164`) should appear in this list, confirming successful dual-interface registration.

**Screenshot: ENI status confirmed as `in-use` and instance health checks passing**

![Step 6a - ENI Status and Instance Health](https://github.com/user-attachments/assets/1a2c8142-8257-4b03-b496-d9ce358d359d)

**Screenshot: Both ENIs confirmed on the instance**

![Step 6b - Dual ENI Registration Confirmed](https://github.com/user-attachments/assets/3a3335b9-b1f1-4f83-9fc1-fb0a69a699b7)

---

## Validation and Final Outcome

| Checkpoint | Expected Result | Status |
|---|---|---|
| AWS credentials active | Valid session with non-expired token | Confirmed |
| Region configured | `us-east-1` | Confirmed |
| EC2 instance identified | `i-0c433b269b876a504` | Confirmed |
| ENI identified | `eni-0642d9bc26cae7164` | Confirmed |
| ENI attached at device index 1 | `AttachmentId` returned | Confirmed |
| ENI status | `in-use` | Confirmed |
| Instance health | `InstanceStatus: ok`, `SystemStatus: ok` | Confirmed |
| Dual ENI registration | Both ENIs listed on instance | Confirmed |

The ENI was successfully attached to the running EC2 instance within the `us-east-1` region. The instance remained in a healthy running state throughout the operation with no disruption to the primary network interface.

---

## Operational Considerations

- **Hot-attach capability:** EC2 supports ENI attachment to running instances without requiring a reboot. However, the guest OS may require manual interface activation (e.g., `ifup eth1` on Linux) after attachment depending on the AMI configuration.
- **ENI portability:** ENIs retain their private IP addresses, Elastic IPs, MAC addresses, and security group memberships when detached and reattached. This makes them ideal for failover architectures.
- **Device index limits:** The maximum number of ENIs per instance varies by instance type. Always verify limits before attachment using `aws ec2 describe-instance-types`.
- **Availability Zone constraint:** An ENI can only be attached to an EC2 instance in the same Availability Zone. Cross-AZ attachment attempts will fail.
- **IAM least privilege:** In production, scope the IAM policy for this operation to the specific instance and ENI ARNs rather than granting broad `ec2:*` permissions.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| Empty query result for instance or ENI | Tag name mismatch or wrong region | Verify tag value and confirm `aws configure get region` |
| `InvalidNetworkInterfaceID.NotFound` | ENI does not exist in the region | Confirm ENI was provisioned in `us-east-1` |
| `AttachmentLimitExceeded` | Instance type ENI limit reached | Check instance type ENI limits and detach unused interfaces |
| `InvalidParameterValue: device index` | Conflicting or out-of-range device index | Check current interfaces and use the next available index |
| ENI status remains `available` after command | API eventual consistency delay | Wait 10-15 seconds and re-query status |
| Instance not reachable after attachment | OS did not auto-configure secondary interface | SSH into instance and bring up the interface manually |

---

## Tags

`aws` `ec2` `eni` `networking` `vpc` `devops` `infrastructure` `cloud-migration` `aws-cli`


