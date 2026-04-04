# Azure VM Disk Expansion and Data Disk Provisioning

![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-28a745?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Configuration](#environment-configuration)
- [Phase 1 - OS Disk Expansion](#phase-1---os-disk-expansion)
- [Phase 2 - Data Disk Provisioning and Attachment](#phase-2---data-disk-provisioning-and-attachment)
- [Phase 3 - In-Guest Disk Configuration](#phase-3---in-guest-disk-configuration)
- [Phase 4 - Validation and Verification](#phase-4---validation-and-verification)
- [Troubleshooting](#troubleshooting)
- [Reference](#reference)

---

## Overview

This runbook documents the end-to-end procedure for expanding an existing Azure Virtual Machine OS disk and provisioning a new managed data disk. It covers all layers of the operation: Azure control plane actions via the Azure CLI, in-guest Linux disk partitioning and formatting, and persistent mount configuration via `/etc/fstab`.

This procedure was executed against a production lab environment on Ubuntu 22.04 LTS (Azure-optimized kernel `6.8.0-1044-azure`) in the `eastus` region.

---

## Problem Statement

### Context

The Nautilus DevOps team required additional storage capacity on an existing virtual machine (`devops-vm`) to support increased application workloads. The existing OS disk was undersized and a dedicated data disk was needed for workload separation.

### Requirements

| Requirement | Target |
|---|---|
| Expand existing OS disk | 32 GiB to 64 GiB |
| Create new managed data disk | 64 GiB, Standard HDD (`Standard_LRS`) |
| Disk name | `devops-disk` |
| Mount point inside VM | `/mnt/devops-disk` |
| Mount persistence | Survive reboots via `/etc/fstab` |

### Root Cause

Azure managed disks cannot be resized while the VM is in a running state. The OS disk had no explicit size tag set in its metadata (a known lab environment behavior), requiring direct JSON inspection to confirm disk state rather than relying on `--query diskSizeGb` output.

---

## Architecture

```
Azure Subscription: Azure Free Labs
   Resource Group: KML_RG_MAIN-866F9CE5075C4378
   Region: eastus
      |
      +-- Virtual Machine: devops-vm (Ubuntu 22.04 LTS)
             |
             +-- OS Disk: devops-vm_disk1_83afd3fc... [Standard_LRS] [64 GiB] <-- EXPANDED
             |
             +-- Temp Disk: /dev/sdb [4 GiB] --> /mnt (Azure ephemeral, NOT modified)
             |
             +-- Data Disk: devops-disk [Standard_LRS] [64 GiB] <-- NEW
                    |
                    +-- Partition: /dev/sdc1 [64 GiB, Linux type]
                    +-- Filesystem: ext4
                    +-- Mount Point: /mnt/devops-disk
                    +-- fstab UUID: bc96015c-5195-4175-81b1-71cb4c0c2b85
```

---

## Prerequisites

### Tools Required

| Tool | Version Used | Purpose |
|---|---|---|
| Azure CLI (`az`) | Latest | Control plane operations |
| `fdisk` | util-linux 2.37.2 | Disk partitioning |
| `mkfs.ext4` | mke2fs 1.46.5 | Filesystem creation |
| `blkid` | Standard Linux | UUID retrieval |
| SSH client | Any | VM access |

### Access Requirements

- Azure CLI authenticated session with Contributor role on the target resource group
- SSH access to `devops-vm` as `azureuser`
- Outbound network access from the `azure-client` host to Azure APIs

---

## Environment Configuration

Establish and validate all shell variables before executing any disk operations. Every subsequent command in this runbook depends on these variables being correctly resolved.

```bash
# Authenticate to Azure
az login --username "<USERNAME>" --password "<PASSWORD>"

# Resolve environment variables dynamically
RESOURCE_GROUP=$(az vm list --query "[0].resourceGroup" -o tsv)
VM_NAME="devops-vm"
LOCATION=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "location" -o tsv)

# Verify all three variables before proceeding
echo "RG: $RESOURCE_GROUP | VM: $VM_NAME | Location: $LOCATION"
```

**Expected output:**
```
RG: KML_RG_MAIN-866F9CE5075C4378 | VM: devops-vm | Location: eastus
```

### Screenshot: Environment Variables Verified

<img width="1032" height="647" alt="image" src="https://github.com/user-attachments/assets/ca1742ac-76ed-41cb-9261-8d4c2eafefc3" />

> *Caption: Terminal output confirming RESOURCE_GROUP, VM_NAME, and LOCATION resolved correctly*

---

## Phase 1 - OS Disk Expansion

### Step 1.1 - Identify the Existing OS Disk

```bash
DISK_NAME=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "storageProfile.osDisk.name" \
  -o tsv)

echo "Disk name: $DISK_NAME"

# Verify disk state using full table output (diskSizeGb may return null in lab environments)
az disk show \
  --resource-group $RESOURCE_GROUP \
  --name $DISK_NAME \
  --query "{Name:name, Size:diskSizeGb, State:diskState, SKU:sku.name}" \
  -o table
```

**Expected output:**
```
Name                                              State     SKU
------------------------------------------------  --------  ------------
devops-vm_disk1_83afd3fcf6da47758c818d0019e38a39  Attached  Standard_LRS
```

> **Known Issue:** In Azure lab environments, `--query "diskSizeGb" -o tsv` may return no output due to a null field in disk metadata. This is cosmetic only. Confirm disk existence and `State: Attached` from the table output and proceed. The actual disk size is confirmed in the `az disk update` JSON response in Step 1.3.

### Screenshot: OS Disk Identified

<img width="1029" height="652" alt="image" src="https://github.com/user-attachments/assets/50b193a3-0b2b-48be-93fa-a74715e9aa34" />

> *Caption: az disk show table output confirming disk name, Attached state, and Standard_LRS SKU*

---

### Step 1.2 - Deallocate the VM

**Critical:** Azure requires the VM to be fully deallocated (not merely stopped) before resizing a managed disk. A stopped-but-not-deallocated VM will cause the resize operation to fail.

```bash
az vm deallocate \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Confirm deallocation before proceeding
az vm get-instance-view \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "instanceView.statuses[1].displayStatus" \
  -o tsv
```

**Expected output:**
```
VM deallocated
```

### Screenshot: VM Deallocated

<img width="1038" height="434" alt="image" src="https://github.com/user-attachments/assets/7a1e4c1b-1c8a-4189-b053-2b2210785e0b" />

> *Caption: Terminal confirming VM deallocated status via get-instance-view*

---

### Step 1.3 - Resize the OS Disk to 64 GiB

```bash
az disk update \
  --resource-group $RESOURCE_GROUP \
  --name $DISK_NAME \
  --size-gb 64
```

**Confirm success from JSON response:**

```
"diskSizeGb": 64,
"provisioningState": "Succeeded"
```

### Screenshots: Disk Resize JSON Response

<img width="1033" height="852" alt="image" src="https://github.com/user-attachments/assets/13b46dc1-4928-4e9f-86a9-89b57a7a6162" />
<img width="1031" height="857" alt="image" src="https://github.com/user-attachments/assets/9071f6e0-4784-43e0-95a6-618964c8dd67" />
<img width="1036" height="864" alt="image" src="https://github.com/user-attachments/assets/1fb9d896-7aab-445c-96ae-c40389d9d25a" />

> *Caption: Full az disk update JSON response showing diskSizeGb: 64 and provisioningState: Succeeded*

---

### Step 1.4 - Restart the VM

```bash
az vm start \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME

# Confirm VM is running
az vm get-instance-view \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "instanceView.statuses[1].displayStatus" \
  -o tsv
```

**Expected output:**
```
VM running
```

### Screenshot: VM Running After Restart

<img width="1033" height="271" alt="image" src="https://github.com/user-attachments/assets/01879f55-ef9d-4cc8-b475-3c966d68a734" />

> *Caption: Terminal showing VM running status after disk resize and restart*

---

## Phase 2 - Data Disk Provisioning and Attachment

### Step 2.1 - Create the New Managed Disk

Standard HDD in Azure uses the SKU identifier `Standard_LRS`.

```bash
az disk create \
  --resource-group $RESOURCE_GROUP \
  --name "devops-disk" \
  --size-gb 64 \
  --sku Standard_LRS \
  --location $LOCATION
```

**Confirm success from JSON response:**

```
"name": "devops-disk",
"diskSizeGb": 64,
"diskState": "Unattached",
"sku": { "name": "Standard_LRS" },
"provisioningState": "Succeeded"
```

### Screenshots: devops-disk Created

<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/3b44575b-4baa-42b0-9ef0-9892886266e7" />
<img width="1037" height="863" alt="image" src="https://github.com/user-attachments/assets/aa6835c0-c893-4bda-a7a7-5465c483fb65" />

> *Caption: az disk create JSON response confirming devops-disk created as Standard_LRS, 64 GiB, Unattached*

---

### Step 2.2 - Attach the Disk to the VM

```bash
az vm disk attach \
  --resource-group $RESOURCE_GROUP \
  --vm-name $VM_NAME \
  --name "devops-disk"

# Verify attachment
az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --query "storageProfile.dataDisks[].{Name:name, Lun:lun, Size:diskSizeGb}" \
  -o table
```

**Expected output:**
```
Name         Lun    Size
-----------  -----  ------
devops-disk  0      64
```

### Screenshot: Disk Attached to VM

<img width="1037" height="317" alt="image" src="https://github.com/user-attachments/assets/5e9d0ab8-55d5-4bc1-bb8c-97e706d75e93" />

> *Caption: az vm show table output confirming devops-disk attached at LUN 0 with 64 GiB*

---

## Phase 3 - In-Guest Disk Configuration

### Step 3.1 - SSH Into the VM

```bash
PUBLIC_IP=$(az vm show \
  --resource-group $RESOURCE_GROUP \
  --name $VM_NAME \
  --show-details \
  --query "publicIps" \
  -o tsv)

echo "Public IP: $PUBLIC_IP"
ssh azureuser@$PUBLIC_IP
```

Upon login, the MOTD system summary will confirm the OS disk resize was applied at the OS level:

```
Usage of /:   2.6% of 61.84GB
```

---

### Step 3.2 - Identify the New Disk Device

```bash
lsblk
```

**Expected output (annotated):**

```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda       8:0    0   64G  0 disk              <-- OS disk (EXPANDED, do not modify)
sda1      8:1    0 63.9G  0 part /
sdb       8:16   0    4G  0 disk              <-- Azure temp/resource disk (do NOT modify)
sdb1      8:17   0    4G  0 part /mnt
sdc       8:32   0   64G  0 disk              <-- devops-disk (NEW, unpartitioned)
```

**Critical:** Never partition or format `sdb`. This is Azure's ephemeral resource disk and its data does not persist across reboots.

### Screenshots: lsblk Output Identifying sdc

<img width="1036" height="405" alt="image" src="https://github.com/user-attachments/assets/912967c9-131b-4c10-aa88-14d209b7788b" />
<img width="1037" height="858" alt="image" src="https://github.com/user-attachments/assets/0e8d8048-d0b4-4980-ad1b-d82d471c22e2" />
<img width="1036" height="477" alt="image" src="https://github.com/user-attachments/assets/0fd11ad2-1b15-446f-a704-246a21bb4eaf" />

> *Caption: lsblk output inside devops-vm showing sda (OS, 64G), sdb (temp, 4G), sdc (new data disk, 64G, unpartitioned)*

---

### Step 3.3 - Partition the New Disk

```bash
sudo fdisk /dev/sdc
```

Inside the `fdisk` interactive prompt, enter the following commands in sequence:

| Input | Action |
|---|---|
| `n` | New partition |
| `p` | Primary partition type |
| `1` | Partition number 1 |
| `[Enter]` | Accept default first sector (2048) |
| `[Enter]` | Accept default last sector (use full disk) |
| `w` | Write partition table and exit |

**Expected confirmation:**
```
Created a new partition 1 of type 'Linux' and of size 64 GiB.
The partition table has been altered.
Syncing disks.
```

### Screenshot: fdisk Partition Created

<img width="1035" height="752" alt="image" src="https://github.com/user-attachments/assets/a55690d5-4804-4f2f-b914-21aeba943e8d" />

> *Caption: fdisk interactive session confirming new 64 GiB Linux partition created on /dev/sdc*

---

### Step 3.4 - Format the Partition as ext4

```bash
sudo mkfs.ext4 /dev/sdc1
```

**Expected output (key lines):**
```
Creating filesystem with 16776960 4k blocks and 4194304 inodes
Filesystem UUID: bc96015c-5195-4175-81b1-71cb4c0c2b85
Writing superblocks and filesystem accounting information: done
```

Note the `Filesystem UUID` from this output. It is used in the next step.

### Screenshot: ext4 Filesystem Created

<img width="1032" height="321" alt="image" src="https://github.com/user-attachments/assets/e9faded3-e571-48f3-8d31-a350aeb51db4" />

> *Caption: mkfs.ext4 output showing filesystem UUID and all steps completing with done status*

---

### Step 3.5 - Create the Mount Point Directory

```bash
sudo mkdir -p /mnt/devops-disk
```

No output is expected. The `-p` flag ensures no error is thrown if the directory already exists.

---

### Step 3.6 - Capture the Disk UUID

Always mount by UUID rather than device name (e.g., `/dev/sdc`). Device names are not guaranteed to persist across reboots in Azure.

```bash
# Use head -1 to prevent capture of secondary UUIDs from the PTUUID field
DISK_UUID=$(sudo blkid /dev/sdc1 | grep -oP 'UUID="\K[^"]+' | head -1)
echo $DISK_UUID
```

**Expected output:**
```
bc96015c-5195-4175-81b1-71cb4c0c2b85
```
**Screenshot:**

<img width="1030" height="444" alt="image" src="https://github.com/user-attachments/assets/7289593c-c136-41e8-a127-285da5179245" />

> **Known Issue:** Without `| head -1`, the `blkid` grep pattern also captures the `PTUUID` value from the partition table, appending a second UUID-like token to the variable. Always pipe through `head -1` to isolate the filesystem UUID.

---

### Step 3.7 - Configure Persistent Mount via fstab

**Always back up `/etc/fstab` before modifying it.** A malformed fstab entry can prevent the VM from booting.

```bash
# Step 1: Backup
sudo cp /etc/fstab /etc/fstab.backup

# Step 2: Append new entry
echo "UUID=$DISK_UUID   /mnt/devops-disk   ext4   defaults,nofail   0   2" \
  | sudo tee -a /etc/fstab

# Step 3: Verify the entry
tail -3 /etc/fstab
```

**Expected tail output:**
```
UUID=45D2-36F5  /boot/efi       vfat    umask=0077      0 1
/dev/disk/cloud/azure_resource-part1    /mnt    auto    defaults,nofail,...
UUID=bc96015c-5195-4175-81b1-71cb4c0c2b85   /mnt/devops-disk   ext4   defaults,nofail   0   2
```

**fstab Field Reference:**

| Field | Value | Meaning |
|---|---|---|
| UUID | `bc96015c-...` | Stable device identifier |
| Mount point | `/mnt/devops-disk` | Target directory |
| Filesystem | `ext4` | Filesystem type |
| Options | `defaults,nofail` | Mount normally; skip on boot if unavailable |
| Dump | `0` | Not included in dump backups |
| Pass | `2` | fsck check order (2 = after root disk) |

### Screenshot: fstab Entry Verified

<img width="1034" height="334" alt="image" src="https://github.com/user-attachments/assets/daff3545-06a5-44db-a94a-d3eb5a69cf3c" />

> *Caption: tail -3 /etc/fstab showing the new devops-disk UUID entry appended correctly*

---

### Step 3.8 - Mount and Verify

```bash
# Mount all fstab entries
sudo mount -a

# Confirm disk is mounted at correct path with correct size
df -h | grep devops-disk
```

**Expected output:**
```
/dev/sdc1        63G   24K   60G   1% /mnt/devops-disk
```

### Screenshot: Disk Mounted Successfully

<img width="1033" height="314" alt="image" src="https://github.com/user-attachments/assets/77979df8-f3b9-41f6-bf71-1366ddc9164f" />

> *Caption: df -h output showing /dev/sdc1 mounted at /mnt/devops-disk with 63G total and 60G available*

---

## Phase 4 - Validation and Verification

Run the full validation suite to confirm all requirements have been met before closing the task.

```bash
# 1. OS disk expanded to 64G
lsblk | grep sda

# 2. Data disk mounted at correct path with correct filesystem type
mount | grep devops-disk

# 3. fstab entry present for reboot persistence
grep devops-disk /etc/fstab

# 4. Write access confirmed
sudo touch /mnt/devops-disk/test_write.txt \
  && echo "Write test PASSED" \
  || echo "Write test FAILED"
```

### Expected Results

```
# Check 1
sda       8:0    0   64G  0 disk
sda1      8:1    0 63.9G  0 part /

# Check 2
/dev/sdc1 on /mnt/devops-disk type ext4 (rw,relatime)

# Check 3
UUID=bc96015c-5195-4175-81b1-71cb4c0c2b85   /mnt/devops-disk   ext4   defaults,nofail   0   2

# Check 4
Write test PASSED
```

### Screenshot: Full Validation Suite Passed

<img width="1036" height="307" alt="image" src="https://github.com/user-attachments/assets/0d4e15bb-c8e2-4580-92a6-707a664b72ab" />

> *Caption: All four validation checks passing in sequence: lsblk, mount, grep fstab, write test PASSED*

---

## Troubleshooting

### `az disk update` fails with "Conflict"

**Cause:** The VM was not fully deallocated. A stopped VM (via the OS `shutdown` command) is not the same as a deallocated VM in Azure.

**Resolution:**
```bash
az vm deallocate --resource-group $RESOURCE_GROUP --name $VM_NAME
# Wait for: VM deallocated
az vm get-instance-view ... --query "instanceView.statuses[1].displayStatus"
```

---

### `--query "diskSizeGb" -o tsv` returns no output

**Cause:** Known behavior in Azure lab environments where the `diskSizeGb` field is null in disk metadata prior to an explicit resize.

**Resolution:** Use the full `az disk update` or `az disk show` JSON response to confirm `"diskSizeGb": 64`. Do not block on the TSV query output.

---

### `blkid` UUID variable captures extra token

**Cause:** The `grep -oP 'UUID="\K[^"]+'` pattern matches both the filesystem UUID and the partition table UUID (`PTUUID`) if present in the `blkid` output.

**Resolution:** Always pipe through `| head -1`:
```bash
DISK_UUID=$(sudo blkid /dev/sdc1 | grep -oP 'UUID="\K[^"]+' | head -1)
```
***Screenshot:***
<img width="1030" height="398" alt="image" src="https://github.com/user-attachments/assets/3146c1cb-1942-4078-9a52-872ca384a3d2" />

---

### `sudo mount -a` returns an error

**Cause:** Malformed fstab entry (wrong UUID, typo in mount point, or incorrect filesystem type).

**Resolution:**
```bash
# Restore backup and re-examine
sudo cp /etc/fstab.backup /etc/fstab
cat /etc/fstab
# Re-run Step 3.7 carefully
```

---

### VM fails to boot after fstab modification

**Cause:** An fstab entry without `nofail` for a non-critical disk causes systemd to halt boot when the disk is unavailable.

**Resolution:** Always use `defaults,nofail` for data disks. The `nofail` option instructs the boot process to continue even if the disk is not present.

---

## Reference

| Resource | Details |
|---|---|
| Subscription | Azure Free Labs (`f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`) |
| Resource Group | `KML_RG_MAIN-866F9CE5075C4378` |
| Region | `eastus` |
| VM | `devops-vm` (Ubuntu 22.04.5 LTS, kernel `6.8.0-1044-azure`) |
| OS Disk | `devops-vm_disk1_83afd3fcf6da47758c818d0019e38a39` |
| Data Disk | `devops-disk` |
| Data Disk UUID | `bc96015c-5195-4175-81b1-71cb4c0c2b85` |
| Mount Point | `/mnt/devops-disk` |
| Execution Date | 2026-03-05 |

### Azure CLI Documentation

- [az disk update](https://learn.microsoft.com/en-us/cli/azure/disk#az-disk-update)
- [az disk create](https://learn.microsoft.com/en-us/cli/azure/disk#az-disk-create)
- [az vm disk attach](https://learn.microsoft.com/en-us/cli/azure/vm/disk#az-vm-disk-attach)
- [az vm deallocate](https://learn.microsoft.com/en-us/cli/azure/vm#az-vm-deallocate)

---
