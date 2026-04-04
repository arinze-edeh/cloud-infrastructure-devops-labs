# Attaching an Existing Managed Disk to an Azure Virtual Machine

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Environment Details](#environment-details)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Access the Azure Portal](#step-1-access-the-azure-portal)
  - [Step 2: Verify the Virtual Machine](#step-2-verify-the-virtual-machine)
  - [Step 3: Verify the Managed Disk](#step-3-verify-the-managed-disk)
  - [Step 4: Attach the Managed Disk to the VM](#step-4-attach-the-managed-disk-to-the-vm)
  - [Step 5: Validate Disk Attachment](#step-5-validate-disk-attachment)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Lessons Learned](#lessons-learned)
- [Validation Checklist](#validation-checklist)
- [Tags](#tags)

---

## Overview

This document details the process of attaching a pre-provisioned Azure managed disk to a running Linux virtual machine as a data disk, using the Azure Portal. Managed disks are Azure's recommended block storage abstraction, providing high availability, automatic replication, and seamless lifecycle management independent of any VM.

This operation is a foundational storage workflow in Azure compute environments and applies directly to scenarios such as live data migration, persistent volume provisioning for stateful workloads, and disaster recovery readiness.

---

## Problem Statement

In production Azure environments, managed disks are frequently provisioned independently of virtual machines, allowing storage to be pre-allocated, pre-configured, or migrated before being associated with compute. The challenge is to correctly attach an unattached managed disk to a running VM without disrupting active workloads, ensuring the disk is registered as a **data disk** (not an OS disk), and verifying its successful attachment through the portal and disk metadata.

**Goal:** Attach the `devops-disk` managed disk to the `devops-vm` virtual machine as a data disk, with full portal-level verification of the attachment state.

---

## Architecture Summary

```
Azure Subscription: Azure Free Labs
  Resource Group: kml_rg_main-4e...
    Virtual Machine: devops-vm (Standard_B1s, East US, Linux, Running)
      OS Disk:   devops-vm_disk1_e5e16b15d6e949a89f8c (Standard HDD LRS, 30 GiB)
      Data Disk: devops-disk (Standard HDD LRS, 30 GiB) <-- ATTACHED in this operation
```

---

## Environment Details

| Parameter | Value |
|---|---|
| Cloud Provider | Microsoft Azure |
| Region | East US |
| Virtual Machine Name | `devops-vm` |
| VM Size | `Standard_B1s` |
| Operating System | Linux |
| Managed Disk Name | `devops-disk` |
| Disk Size | 30 GiB |
| Storage Type | Standard HDD LRS |
| Resource Group | `kml_rg_main-4e289666b41d4c5b` |

---

## Prerequisites

- Active Azure Portal access with sufficient RBAC permissions (Contributor or higher on the resource group)
- A running virtual machine (`devops-vm`) in `East US`
- An existing managed disk (`devops-disk`) in the same region and subscription as the VM
- The managed disk must be in an **Unattached** state prior to this operation
- VM initialization must be complete before attachment

---

## Implementation

---

### Step 1: Access the Azure Portal

**Intent:** Authenticate to the Azure Portal and confirm access to the target subscription and resource environment.

- OPEN the Azure Portal at [https://portal.azure.com](https://portal.azure.com)
- ENTER the provided lab credentials (username and password)
- CONFIRM successful login and verify the active subscription is `Azure Free Labs` (visible in the top-right account badge)

> **Operational Note:** Always confirm the active subscription context before performing disk operations. Attaching resources across subscriptions is not supported via the portal disk attachment workflow.

**Screenshot: Azure Portal home after successful authentication**

![Azure Portal Login](https://github.com/user-attachments/assets/3830c7b2-363a-47e4-bea4-c5c816c3ff06)

*Azure Portal home screen confirming authenticated access to the Azure Free Labs subscription.*

---

### Step 2: Verify the Virtual Machine

**Intent:** Confirm the target VM exists, is in the correct region, and is in a `Running` state before initiating any disk operations.

- NAVIGATE to **Virtual Machines** from the Azure Portal home or the top search bar
- LOCATE and SELECT `devops-vm` from the VM list
- VERIFY the following attributes in the VM list view:
  - **Name:** `devops-vm`
  - **Resource Group:** `kml_rg_main-4e...`
  - **Location:** `East US`
  - **Status:** `Running`
  - **OS:** `Linux`
  - **Size:** `Standard_B1s`
  - **Public IP:** `40.87.30.237`
  - **Disks:** `1` (OS disk only, prior to attachment)

> **Risk:** Attempting disk attachment to a VM in a `Stopped (Deallocated)` state requires the VM to be started first. While Azure supports hot-attach for data disks on running Linux VMs, the VM must be initialized and healthy before the attachment is validated end-to-end.

**Screenshot: Virtual Machines list showing devops-vm in Running state**

![VM Verification](https://github.com/user-attachments/assets/05e3d5ad-6c1a-43ca-82e9-a1f6f8318c82)

*Compute infrastructure view confirming devops-vm is Running in East US with Standard_B1s sizing.*

---

### Step 3: Verify the Managed Disk

**Intent:** Confirm the `devops-disk` managed disk exists, is located in the correct region, and is in an `Unattached` state, making it eligible for attachment.

- NAVIGATE to **Disks** via the portal search bar or via `Storage Center > Azure Disks`
- LOCATE and SELECT `devops-disk` from the disk list
- VERIFY the following properties in the disk overview:
  - **Disk State:** `Unattached`
  - **Location:** `East US`
  - **Disk Size:** `30 GiB`
  - **Storage Type:** `Standard HDD LRS`
  - **Security Type:** `Standard`
  - **IOPS:** `500`
  - **Throughput:** `60 MBps`
  - **Subscription:** `Azure Free Labs`
  - **Managed By:** `-` (confirms no current VM association)

> **Edge Case:** If the disk state is `Attached`, it is already associated with another VM. You must detach it from the source VM before it can be attached elsewhere. Azure does not support simultaneous attachment of a Standard HDD disk to multiple VMs (Max Shares = 0 for non-shared disks).

**Screenshot: devops-disk overview confirming Unattached state and 30 GiB Standard HDD LRS configuration**

![Disk Verification - Unattached](https://github.com/user-attachments/assets/26a4cc7f-79ca-4314-9a20-de66e74760f5)

*Azure Disks view showing devops-disk in Unattached state, confirming it is eligible for attachment.*

---

### Step 4: Attach the Managed Disk to the VM

**Intent:** Associate the pre-provisioned `devops-disk` with `devops-vm` as a data disk at LUN 0, preserving the existing OS disk and adding block storage capacity for workload use.

- NAVIGATE back to **Virtual Machines** and SELECT `devops-vm`
- In the VM blade left navigation, expand **Settings** and SELECT **Disks**
- In the **Data disks** section, CLICK **Attach existing disks**
- In the **Disk name** dropdown, SELECT `devops-disk`
- CONFIRM the LUN assignment is `0` (first available data disk slot)
- VERIFY the **Storage type** reads `Standard HDD LRS`
- CLICK **Apply** to save the disk configuration change

> **Key Decision:** Attaching as a **data disk** (not swapping the OS disk) ensures the VM's root filesystem remains intact. Data disks are independently managed, can be detached and reattached without affecting the OS, and are the correct target for additional persistent storage.

> **Operational Note:** Azure allows hot-attach of data disks to running Linux VMs without a reboot. However, the disk will need to be initialized, partitioned, formatted, and mounted inside the OS (via `lsblk`, `fdisk`, `mkfs`, and `/etc/fstab`) before it is usable at the filesystem level.

**Screenshot: Apply button active, confirming the disk attachment configuration is staged**

![Attach Disk - Configuration](https://github.com/user-attachments/assets/a929ec8e-dfcf-48ba-bd53-9e3d3f740366)


*Data disk configuration with devops-disk at LUN 0 staged for apply. Apply and Discard changes buttons are active.*

---

### Step 5: Validate Disk Attachment

**Intent:** Confirm the attachment completed successfully at both the VM resource level and the disk resource level, with no errors.

**VM-level validation:**

- WAIT for the green **"Updated virtual machine"** success notification: `Successfully updated virtual machine 'devops-vm'`
- CONFIRM the `devops-disk` now appears in the **Data disks** table under `devops-vm | Disks`
- VERIFY:
  - **LUN:** `0`
  - **Disk name:** `devops-disk`
  - **Storage type:** `Standard HDD LRS`
  - **Size:** `30 GiB`
  - **Max IOPS:** `500`
  - **Max Throughput:** `60 MBps`
  - **Encryption:** `SSE with PMK`

**Screenshot: devops-vm Disks blade confirming devops-disk attached as data disk with success notification**

<img width="1869" height="949" alt="image" src="https://github.com/user-attachments/assets/57cea53b-94fc-4ff0-8444-d5d57f10505c" />

![Disk Attached - Success](https://github.com/user-attachments/assets/c177604c-3614-4571-8452-45bb3d19cc8a)

*Success notification "Updated virtual machine devops-vm" and Data disks table showing devops-disk at LUN 0.*

**Disk-level validation:**

- NAVIGATE to **Disks** and SELECT `devops-disk`
- VERIFY the following fields have updated:
  - **Disk State:** `Attached` (changed from `Unattached`)
  - **Managed By:** `devops-vm` (confirms the VM association)
  - **Last Ownership Update:** timestamp reflecting the attachment event
  - **Provisioning State:** `Succeeded`

**Screenshot: devops-disk overview confirming Attached state and Managed by devops-vm**

![Disk Attached - Disk-Level Validation](https://github.com/user-attachments/assets/c177604c-3614-4571-8452-45bb3d19cc8a)

*Azure Disks view showing devops-disk state changed to Attached with Managed by field updated to devops-vm.*

---

## Key Decisions

| Decision | Rationale | Trade-off |
|---|---|---|
| Attach as **data disk**, not OS disk swap | Preserves the running OS and boot volume; data disks are the correct vehicle for additional storage capacity | Requires OS-level initialization (partition, format, mount) before the disk is usable at the filesystem level |
| Use **Standard HDD LRS** | Cost-effective for dev and non-latency-sensitive workloads; matches the pre-provisioned disk SKU | Not suitable for production IOPS-intensive workloads; use Premium SSD or Ultra Disk for those scenarios |
| Attach via **portal** rather than CLI | Provides visual confirmation of LUN assignment and disk state at each step | CLI (`az vm disk attach`) is preferred for automation, scripting, and IaC pipelines |
| **LUN 0** assignment | First available data disk slot; Azure assigns LUN automatically when unspecified | LUN assignments are significant in multi-disk configurations; document assignments for operational clarity |

---

## Best Practices and Operational Considerations

- **Same region requirement:** Managed disks can only be attached to VMs in the same Azure region. Cross-region attachment requires disk export, copy, and re-import.
- **Post-attachment OS initialization:** Attaching a disk at the Azure resource layer does not make it usable within the OS. After attachment, connect via SSH and run `lsblk` to identify the new device (typically `/dev/sdc`), then partition, format, and mount it. Add a persistent entry to `/etc/fstab` using the disk UUID to survive reboots.
- **Avoid Standard HDD for production:** Standard HDD LRS provides 500 IOPS and 60 MBps throughput, suitable for dev environments. Production workloads with throughput or latency requirements should use Premium SSD LRS or Premium SSD v2.
- **Snapshot before attaching to production VMs:** Always create a disk snapshot prior to attaching or detaching managed disks on production systems to preserve a recovery point.
- **Tag managed disks consistently:** Apply resource tags (`environment`, `owner`, `workload`) to disks at creation time to prevent orphaned disk accumulation, which incurs ongoing storage costs even when unattached.
- **Monitor disk throughput limits against VM size:** The `Standard_B1s` VM size supports a maximum of 23 MBps aggregate disk throughput. Attaching multiple disks rated at 60 MBps each does not increase the VM-level cap. This mismatch is surfaced by the Azure portal warning visible in the Disks blade.

---

## Risks, Edge Cases, and Troubleshooting

**Risk: Disk in Attached state**
If `devops-disk` shows `Attached` before this operation, it is owned by another VM. Navigate to the owning VM, go to `Settings > Disks`, select the disk row, click the detach icon, and save. Wait for the disk state to return to `Unattached` before proceeding.

**Risk: Disk and VM in different regions**
Azure will not surface the disk in the attachment dropdown if the disk and VM are in different regions. Confirm both resources are in `East US` before beginning.

**Risk: VM-level disk throughput cap**
The `Standard_B1s` VM size caps total disk throughput at 23 MBps regardless of the attached disks' individual ratings. If workloads require higher throughput, resize the VM to a larger SKU that supports higher aggregate disk throughput.

**Risk: Disk not visible inside the OS after attachment**
If the disk does not appear in `lsblk` output after hot-attach, try running `echo "- - -" | sudo tee /sys/class/scsi_host/host*/scan` to trigger a rescan of the SCSI bus without rebooting.

**Risk: Apply button grayed out**
If the Apply button remains inactive after selecting the disk, refresh the portal blade and reattempt. This is occasionally caused by transient portal state caching.

---

## Lessons Learned

- The Azure Portal surfaces a throughput mismatch warning when attached disk throughput ratings exceed the VM SKU's aggregate cap. This is informational and does not block the attachment, but it is an important signal for right-sizing VM compute alongside storage.
- Disk state changes (`Unattached` to `Attached`) are reflected immediately in the disk overview after a successful VM update, providing a reliable second confirmation point independent of the VM blade.
- Standard HDD OS disks will be retired by Azure on **September 8, 2028**. This banner is visible in the Disks blade as an informational notice. Plan migrations to Standard SSD or Premium SSD OS disks ahead of this deprecation date.
- Pre-provisioning managed disks independently of VMs is a valid and recommended pattern for storage readiness workflows, enabling disk pre-configuration (encryption, sizing, SKU selection) before compute is available.

---

## Validation Checklist

- [ ] Azure Portal access confirmed with correct subscription context (`Azure Free Labs`)
- [ ] `devops-vm` verified as `Running` in `East US` with `Standard_B1s` sizing
- [ ] `devops-disk` verified as `Unattached` in `East US` with 30 GiB `Standard HDD LRS`
- [ ] `devops-disk` attached to `devops-vm` via `Settings > Disks > Attach existing disks`
- [ ] Attachment applied at LUN 0 as a data disk
- [ ] Success notification `"Successfully updated virtual machine 'devops-vm'"` received
- [ ] `devops-disk` visible in the Data disks table of `devops-vm | Disks`
- [ ] `devops-disk` overview shows `Disk state: Attached` and `Managed by: devops-vm`
- [ ] No deployment or configuration errors observed
- [ ] VM initialization confirmed complete

---

## Tags

`azure` `compute` `virtual-machine` `managed-disk` `storage` `devops` `cloud-operations` `block-storage` `data-disk` `azure-portal`























# Attach Existing Managed Disk to Azure Virtual Machine


## Objective
- Attach an existing Azure managed disk to an existing virtual machine
- Ensure the disk is attached as a **data disk**
- Validate that the virtual machine initialization is complete

---

## Environment Details
- Cloud Provider: `Azure`
- Region: `eastus`
- Virtual Machine Name: `devops-vm`
- Managed Disk Name: `devops-disk`

---

## Prerequisites
- Azure Portal access
- Existing virtual machine
- Existing managed disk
- VM and disk must be in the same region
- VM must be initialized before submission

---

## Step-by-Step Implementation

---

### Step 1: Login to Azure Portal
- OPEN Azure Portal using Portal URL
- ENTER username
- ENTER password
- CONFIRM successful login

screenshot: `azure-portal-login`

<img width="1828" height="946" alt="image" src="https://github.com/user-attachments/assets/3830c7b2-363a-47e4-bea4-c5c816c3ff06" />

---

### Step 2: Verify Virtual Machine
- NAVIGATE to `Virtual Machines`
- SELECT `devops-vm`
- VERIFY:
  - VM exists
  - Region = `eastus`
  - Status = `Running`

screenshot: `vm-overview`
<img width="1830" height="950" alt="image" src="https://github.com/user-attachments/assets/05e3d5ad-6c1a-43ca-82e9-a1f6f8318c82" />

---

### Step 3: Verify Managed Disk
- NAVIGATE to `Disks`
- SELECT `devops-disk`
- VERIFY:
  - Disk exists
  - Region = `eastus`
  - Disk is not attached to any VM

screenshot: `disk-overview`
<img width="1855" height="946" alt="image" src="https://github.com/user-attachments/assets/26a4cc7f-79ca-4314-9a20-de66e74760f5" />

---

### Step 4: Attach Managed Disk to VM
- OPEN virtual machine `devops-vm`
- NAVIGATE to `Settings → Disks`
- CLICK `Attach existing disk`
- SELECT disk `devops-disk`
- SET disk type = Data disk
- CLICK `Save`

screenshot: `attach-disk-configuration`
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/a929ec8e-dfcf-48ba-bd53-9e3d3f740366" />
<img width="1869" height="949" alt="image" src="https://github.com/user-attachments/assets/ebb5dec2-5d87-493c-9aac-e89486f5c6dc" />

---

### Step 5: Validate Disk Attachment
- WAIT for deployment to complete
- CONFIRM:
  - Disk appears under Data disks
  - Disk status = Attached

screenshot: `disk-attached-success`
<img width="1822" height="952" alt="image" src="https://github.com/user-attachments/assets/c177604c-3614-4571-8452-45bb3d19cc8a" />

---

### Validation Checklist
- VM `devops-vm` is running
- Disk `devops-disk` is attached as a data disk
- No deployment or configuration errors
- VM initialization completed

---

### Tags
`azure` `compute`
`virtual-machine`
`managed-disk`
`storage`
`devops`
`cloud-operations`
