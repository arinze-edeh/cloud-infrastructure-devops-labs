# AWS EC2 EBS Root Volume Expansion -- Live Instance (Zero Downtime)

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20EBS-orange?logo=amazonaws)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen)
![Region](https://img.shields.io/badge/Region-us--east--1-blue)
![OS](https://img.shields.io/badge/OS-Amazon%20Linux%202023-yellow)
![Disk](https://img.shields.io/badge/Volume-gp3%20%7C%208GB%20to%2012GB-lightgrey)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Environment Overview](#environment-overview)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Step 1 -- Identify the Target Instance](#step-1----identify-the-target-instance)
  - [Step 2 -- Inspect the Attached EBS Volume](#step-2----inspect-the-attached-ebs-volume)
  - [Step 3 -- Expand the EBS Volume via AWS API](#step-3----expand-the-ebs-volume-via-aws-api)
  - [Step 4 -- Monitor Volume Modification Progress](#step-4----monitor-volume-modification-progress)
  - [Step 5 -- Retrieve Instance Public Endpoint](#step-5----retrieve-instance-public-endpoint)
  - [Step 6 -- Secure the Key Pair and SSH into the Instance](#step-6----secure-the-key-pair-and-ssh-into-the-instance)
  - [Step 7 -- Verify Disk State Inside the Instance (lsblk)](#step-7----verify-disk-state-inside-the-instance-lsblk)
  - [Step 8 -- Confirm Filesystem Has NOT Yet Expanded (df)](#step-8----confirm-filesystem-has-not-yet-expanded-df)
  - [Step 9 -- Inspect Partition Table (fdisk)](#step-9----inspect-partition-table-fdisk)
  - [Step 10 -- Grow the Partition to Fill the New Volume Size](#step-10----grow-the-partition-to-fill-the-new-volume-size)
  - [Step 11 -- Confirm Partition Expansion (lsblk)](#step-11----confirm-partition-expansion-lsblk)
  - [Step 12 -- Expand the XFS Filesystem Online](#step-12----expand-the-xfs-filesystem-online)
  - [Step 13 -- Validate Final Disk Usage](#step-13----validate-final-disk-usage)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Quick Reference Cheatsheet](#quick-reference-cheatsheet)
- [Author](#author)

---

## Problem Statement

The Development Team reported that the `datacenter-ec2` EC2 instance was running critically low on storage. The root EBS volume was provisioned at **8 GiB** and needed to be expanded to **12 GiB** to accommodate growing data requirements.

**Requirements:**
- Zero downtime -- the instance must remain running throughout the operation
- The expanded storage must be immediately usable inside the instance (no reboot)
- All work must be isolated to the `us-east-1` region

---

## Environment Overview

| Parameter | Value |
|-----------|-------|
| Instance Name | `datacenter-ec2` |
| Instance ID | `i-0a39964f2f9a12e81` |
| Region | `us-east-1` |
| Volume ID | `vol-0dd324452a07f97f1` |
| Volume Type | `gp3` |
| Original Size | `8 GiB` |
| Target Size | `12 GiB` |
| Device Path | `/dev/xvda` |
| Root Partition | `/dev/xvda1` |
| Filesystem | `XFS` |
| Operating System | Amazon Linux 2023 |
| Instance State | `running` (live, no reboot) |
| Public IP | `54.226.22.120` |

---

## Architecture Context

```
+---------------------+         +---------------------------+
|   aws-client host   |  SSH    |   datacenter-ec2          |
|   (management)      +-------->|   i-0a39964f2f9a12e81     |
|   us-east-1         |         |   Amazon Linux 2023        |
+----------+----------+         +------------+--------------+
           |                                 |
           | AWS CLI (modify-volume)         | /dev/xvda (gp3)
           |                                 |   xvda1 (root, XFS)
           v                                 |   xvda127 (BIOS boot)
    +------+------+                          |   xvda128 (EFI)
    |  AWS EBS    |<------------------------+
    |  API        |   8 GiB --> 12 GiB
    +-------------+
```

---

## Prerequisites

- AWS CLI configured with sufficient IAM permissions (`ec2:DescribeInstances`, `ec2:DescribeVolumes`, `ec2:ModifyVolume`, `ec2:DescribeVolumesModifications`)
- SSH key pair file: `/root/datacenter-keypair.pem`
- Network access to the instance public IP on port 22
- `growpart` utility available inside the instance (part of `cloud-utils-growpart`)
- `xfs_growfs` utility available (part of `xfsprogs`)

---

## Resolution Walkthrough

### Step 1 -- Identify the Target Instance

Retrieve the instance ID and its attached EBS volume IDs by filtering on the `Name` tag.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].[InstanceId,State.Name,BlockDeviceMappings[*].Ebs.VolumeId]" \
  --output table
```

**Output:**

```
---------------------------
|    DescribeInstances    |
+-------------------------+
|  i-0a39964f2f9a12e81    |
|  running                |
|  vol-0dd324452a07f97f1  |
+-------------------------+
```

> **Key Observation:** Instance is in `running` state. The attached volume is `vol-0dd324452a07f97f1`. This confirms the correct target before any destructive or modifying actions are taken.


**SCREENSHOT**

<img width="1030" height="519" alt="image" src="https://github.com/user-attachments/assets/b25e379c-7f00-4a1b-acbf-98e28d5f3c75" />

---

### Step 2 -- Inspect the Attached EBS Volume

Retrieve detailed volume metadata including current size, state, type, and mount device.

```bash
aws ec2 describe-volumes \
  --filters "Name=attachment.instance-id,Values=i-0a39964f2f9a12e81" \
  --region us-east-1 \
  --query "Volumes[*].[VolumeId,Size,State,VolumeType,Attachments[0].Device]" \
  --output table
```

**Output:**

```
--------------------------------------------------------------
|                       DescribeVolumes                      |
+------------------------+----+---------+------+-------------+
|  vol-0dd324452a07f97f1 |  8 |  in-use |  gp3 |  /dev/xvda  |
+------------------------+----+---------+------+-------------+
```

> **Key Observation:** Volume is `8 GiB`, type `gp3`, attached at `/dev/xvda`, and currently `in-use`. This is the root device. Confirmed baseline state before modification.


**SCREENSHOT** 

<img width="1034" height="604" alt="image" src="https://github.com/user-attachments/assets/359c405d-69d0-49cf-b66b-58d140690fdc" />


---

### Step 3 -- Expand the EBS Volume via AWS API

Modify the volume size from 8 GiB to 12 GiB. This operation is non-disruptive -- AWS expands the volume while it remains attached and the instance keeps running.

```bash
aws ec2 modify-volume \
  --volume-id vol-0dd324452a07f97f1 \
  --size 12 \
  --region us-east-1
```

**Output:**

```json
{
    "VolumeModification": {
        "VolumeId": "vol-0dd324452a07f97f1",
        "ModificationState": "modifying",
        "TargetSize": 12,
        "TargetIops": 3000,
        "TargetVolumeType": "gp3",
        "TargetThroughput": 125,
        "TargetMultiAttachEnabled": false,
        "OriginalSize": 8,
        "OriginalIops": 3000,
        "OriginalVolumeType": "gp3",
        "OriginalThroughput": 125,
        "OriginalMultiAttachEnabled": false,
        "Progress": 0,
        "StartTime": "2026-03-26T00:46:13.000Z"
    }
}
```

> **Key Observation:** `ModificationState` is `modifying` and `Progress` is `0`. The AWS EBS modification pipeline has been initiated. IOPS and throughput remain unchanged (`3000 IOPS`, `125 MB/s`) since only size was modified.


**SCREENSHOT** 

<img width="1026" height="692" alt="image" src="https://github.com/user-attachments/assets/1ba8b60e-1117-4c25-a958-015661296c2b" />

---

### Step 4 -- Monitor Volume Modification Progress

Poll the modification status to confirm the volume has successfully completed expansion before proceeding with OS-level changes.

```bash
aws ec2 describe-volumes-modifications \
  --volume-ids vol-0dd324452a07f97f1 \
  --region us-east-1 \
  --query "VolumesModifications[*].[VolumeId,ModificationState,TargetSize,Progress]" \
  --output table
```

**Output:**

```
----------------------------------------------------
|           DescribeVolumesModifications           |
+------------------------+--------------+-----+----+
|  vol-0dd324452a07f97f1 |  optimizing  |  12 |  0 |
+------------------------+--------------+-----+----+
```

> **Key Observation:** `ModificationState` has transitioned from `modifying` to `optimizing`. This means the volume resize is **complete** at the storage layer. The `optimizing` state indicates AWS is fine-tuning I/O performance but the full 12 GiB is already available. It is now safe to proceed with OS-level partition and filesystem operations.


**SCREENSHOT** 

<img width="1033" height="692" alt="image" src="https://github.com/user-attachments/assets/e2df4b0b-5574-4652-a193-1f9077dd84a3" />


---

### Step 5 -- Retrieve Instance Public Endpoint

Fetch the public IP and public DNS name to use for SSH access.

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=datacenter-ec2" \
  --region us-east-1 \
  --query "Reservations[*].Instances[*].[PublicIpAddress,PublicDnsName]" \
  --output table
```

**Output:**

```
----------------------------------------------------------------
|                       DescribeInstances                      |
+----------------+---------------------------------------------+
|  54.226.22.120 |  ec2-54-226-22-120.compute-1.amazonaws.com  |
+----------------+---------------------------------------------+
```

> **Key Observation:** Public IP is `54.226.22.120`. DNS is `ec2-54-226-22-120.compute-1.amazonaws.com`. Both can be used for SSH. IP is preferred for scripted access; DNS is preferred for resilience against IP changes.

**SCREENSHOT** 

<img width="1031" height="869" alt="image" src="https://github.com/user-attachments/assets/2f2f02c1-fd45-4dc1-8182-b7e7eda16d64" />

---

### Step 6 -- Secure the Key Pair and SSH into the Instance

Set restrictive permissions on the key pair file (required by SSH), then establish a connection.

```bash
chmod 400 /root/datacenter-keypair.pem

ssh -i /root/datacenter-keypair.pem ec2-user@54.226.22.120
```

**Output (condensed):**

```
The authenticity of host '54.226.22.120 (54.226.22.120)' can't be established.
ECDSA key fingerprint is SHA256:MRyQ9jkap/t43JHq2GRkij2WEH/DuUq19djztFfCvfA.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '54.226.22.120' (ECDSA) to the list of known hosts.

   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ...
[ec2-user@ip-172-31-30-151 ~]$
```

> **Key Observation:** `chmod 400` is mandatory. SSH will refuse to use a key file with permissions wider than `400` (owner read-only). The host key fingerprint warning is expected on first connection. Confirming `yes` adds the fingerprint to `~/.ssh/known_hosts`.

> **Note:** Multiple newer Amazon Linux 2023 versions were listed upon login. This is informational. Updating the OS is outside the scope of this runbook but should be tracked as a follow-up action.


**SCREENSHOTS**

<img width="1028" height="863" alt="image" src="https://github.com/user-attachments/assets/15a271d2-6384-463a-9827-2a4d581e3fd0" />
<img width="1032" height="859" alt="image" src="https://github.com/user-attachments/assets/888df0e6-4e4e-46f4-88c5-b755c77b0b0f" />

> Terminal showing successful SSH login to `ec2-user@ip-172-31-30-151` with the Amazon Linux 2023 banner.

---

### Step 7 -- Verify Disk State Inside the Instance (lsblk)

Confirm that the kernel has detected the expanded block device size.

```bash
lsblk
```

**Output:**

```
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0  12G  0 disk
├─xvda1   202:1    0   8G  0 part /
├─xvda127 259:0    0   1M  0 part
└─xvda128 259:1    0  10M  0 part /boot/efi
```

> **Key Observation:** The disk `xvda` now shows `12G` -- the kernel sees the full new volume size. However, the root partition `xvda1` still shows `8G`. This is the critical gap: **the block device is expanded but the partition boundary has not yet been moved**. The filesystem also remains at 8G. Two more steps are required.


**SCREENSHOT**

<img width="1029" height="457" alt="image" src="https://github.com/user-attachments/assets/1584b123-2095-4895-a8fd-94d09ce13f1b" />

>Terminal output of `lsblk` showing `xvda` at `12G` but `xvda1` still at `8G`.

---

### Step 8 -- Confirm Filesystem Has NOT Yet Expanded (df)

Verify the filesystem reported size before any partition or filesystem operation.

```bash
df -hT /
```

**Output:**

```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/xvda1     xfs   8.0G  1.6G  6.5G  19% /
```

> **Key Observation:** The XFS filesystem is still reporting `8.0G`. This is expected at this stage. The AWS-side expansion is complete, but the partition and filesystem inside the OS have not yet been resized. Proceeding to `growpart` will resolve the partition boundary issue.

**SCREENSHOT**

<img width="1031" height="421" alt="image" src="https://github.com/user-attachments/assets/7a4d4d29-7231-404d-a37d-6c2a2d947774" />

---

### Step 9 -- Inspect Partition Table (fdisk)

Examine the current partition layout and confirm the GPT PMBR size mismatch warning (expected and safe to proceed with).

```bash
sudo fdisk -l
```

**Output:**

```
GPT PMBR size mismatch (16777215 != 25165823) will be corrected by write.
The backup GPT table is not on the end of the device.
Disk /dev/xvda: 12 GiB, 12884901888 bytes, 25165824 sectors
...
Device       Start      End  Sectors Size Type
/dev/xvda1   24576 16777182 16752607   8G Linux filesystem
/dev/xvda127 22528    24575     2048   1M BIOS boot
/dev/xvda128  2048    22527    20480  10M EFI System

Partition table entries are not in disk order.
```

> **Key Observation:** Two warnings appear here. Both are expected and non-blocking:
> 1. `GPT PMBR size mismatch` -- The protective MBR in the GPT still references the old 8G size. This is automatically corrected on the next partition write by `growpart`.
> 2. `Partition table entries are not in disk order` -- The EFI and BIOS boot partitions are placed before the data partition in physical layout (common for AWS AMIs). This does not affect functionality.

> `xvda1` currently ends at sector `16777182`, well short of the disk end at sector `25165823`. The unallocated space is available for partition extension.

> **SCREENSHOT**

<img width="1035" height="718" alt="image" src="https://github.com/user-attachments/assets/b75c1bed-b77c-4a12-a7b9-8b211bf49abc" />

> Terminal output of `sudo fdisk -l` highlighting the GPT PMBR mismatch warning and the old partition end boundary for `xvda1`.

---

### Step 10 -- Grow the Partition to Fill the New Volume Size

Use `growpart` to extend partition 1 (`xvda1`) to consume all available unallocated space on the disk.

```bash
sudo growpart /dev/xvda 1
```

**Output:**

```
CHANGED: partition=1 start=24576 old: size=16752607 end=16777183 new: size=25141215 end=25165791
```

> **Key Observation:** Partition 1 has been successfully extended. The new end sector is `25165791`, effectively claiming the entire usable disk. The `CHANGED` status confirms the GPT was written and the PMBR mismatch was corrected. The filesystem itself is still 8G -- that is addressed in the next step.


**SCREENSHOT** 

<img width="1036" height="549" alt="image" src="https://github.com/user-attachments/assets/73c721b7-52d6-4fca-844b-56be020047ff" />

>Terminal output showing `CHANGED: partition=1` with old and new size values confirming successful partition extension.

---

### Step 11 -- Confirm Partition Expansion (lsblk)

Verify the partition now reflects the full 12G before expanding the filesystem.

```bash
lsblk
```

**Output:**

```
NAME      MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
xvda      202:0    0  12G  0 disk
├─xvda1   202:1    0  12G  0 part /
├─xvda127 259:0    0   1M  0 part
└─xvda128 259:1    0  10M  0 part /boot/efi
```

> **Key Observation:** `xvda1` now shows `12G`. Disk, partition, and mount are all aligned at 12G at the block device level. The final step is to notify the XFS filesystem driver of the new partition size.


> **SCREENSHOT**

<img width="1034" height="556" alt="image" src="https://github.com/user-attachments/assets/36e796d9-4d82-4129-bd56-81c823311a24" />

>Terminal output of `lsblk` showing both `xvda` and `xvda1` now reporting `12G`.

---

### Step 12 -- Expand the XFS Filesystem Online

Use `xfs_growfs` to expand the XFS filesystem to fill the newly enlarged partition. This is a fully online, non-disruptive operation -- no unmount required.

```bash
sudo xfs_growfs -d /
```

**Output:**

```
meta-data=/dev/xvda1             isize=512    agcount=2, agsize=1047040 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1
data     =                       bsize=4096   blocks=2094075, imaxpct=25
         =                       sunit=128    swidth=128 blks
naming   =version 2              bsize=16384  ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=4096  sunit=4 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
data blocks changed from 2094075 to 3142651
```

> **Key Observation:** `data blocks changed from 2094075 to 3142651` -- this is the definitive confirmation that the XFS filesystem has been successfully expanded. The `-d` flag targets the data subvolume (the root). The filesystem was resized **online** with no service interruption.

<!-- SCREENSHOT PLACEHOLDER -->
> **[SCREENSHOT 10]** -- Terminal output of `sudo xfs_growfs -d /` showing `data blocks changed from 2094075 to 3142651`.

---

### Step 13 -- Validate Final Disk Usage

Confirm the filesystem now reports the full 12G as available capacity.

```bash
df -hT /
```

**Output:**

```
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/xvda1     xfs    12G  1.6G   11G  13% /
```

> **Key Observation:** Success. The root filesystem now shows `12G` total with `11G` available. Usage dropped from `19%` (pre-expansion) to `13%`, providing headroom for continued development workloads. The entire operation was completed with **zero downtime**.

<!-- SCREENSHOT PLACEHOLDER -->
> **[SCREENSHOT 11]** -- Terminal output of `df -hT /` showing filesystem size as `12G` with `11G` available, confirming full end-to-end resolution.

---

## Errors Encountered and Resolutions

### Warning 1 -- GPT PMBR Size Mismatch

**Observed during:** `sudo fdisk -l` (Step 9)

**Warning message:**
```
GPT PMBR size mismatch (16777215 != 25165823) will be corrected by write.
The backup GPT table is not on the end of the device.
```

**Root cause:** When AWS expanded the EBS volume at the storage layer, the physical disk size changed from 8G to 12G. The GPT protective MBR (PMBR) still stored the old sector count (`16777215`). This is a metadata inconsistency in the partition table, not a filesystem error.

**Resolution:** This warning is **expected and self-healing**. `growpart` automatically corrected both the PMBR and the backup GPT table during the partition write in Step 10. No manual intervention was required.

**Severity:** Informational only. No data risk.

---

### Warning 2 -- Partition Table Entries Not in Disk Order

**Observed during:** `sudo fdisk -l` (Step 9)

**Warning message:**
```
Partition table entries are not in disk order.
```

**Root cause:** This is a characteristic of AWS-managed AMI partition layouts. The EFI System partition (`xvda128`) and BIOS boot partition (`xvda127`) are physically located before `xvda1` in sector space but have higher partition numbers. This does not affect runtime behavior or the expansion process.

**Resolution:** No action required. This is by design in AWS AMIs and does not impact the `growpart` or `xfs_growfs` operations.

**Severity:** Informational only.

---

### Implicit Issue -- Two-Layer Expansion Requirement

**Observed during:** Steps 7 and 8

**Description:** After completing the AWS-side volume expansion (Steps 3 and 4), many engineers mistakenly assume the storage is immediately usable. As shown in Step 7 and Step 8, the OS-level partition (`xvda1`) and filesystem (XFS) still reported 8G even after the volume was expanded to 12G at the AWS layer.

**Root cause:** AWS EBS operates at the block device layer. The OS partition table and filesystem are completely separate layers that must each be explicitly updated.

**Resolution:** A mandatory two-step OS-side process was executed:
1. `growpart` to extend the partition boundary
2. `xfs_growfs` to expand the filesystem into the new partition space

**Severity:** Operational gap -- skipping these steps would result in the expanded storage being invisible to the OS.

---

## Best Practices

### Pre-Change

* **Always snapshot before expanding** -- Create an EBS snapshot before any volume modification. Even though this is a non-destructive operation, a snapshot provides a rollback point with minimal cost.
* **Verify the correct volume is targeted** -- Use `describe-volumes` with the instance ID filter (not just the volume ID in isolation) to confirm attachment context before modifying.
* **Check for pending modifications** -- Run `describe-volumes-modifications` before initiating a new modification. AWS enforces a cooldown: you cannot modify a volume again within 6 hours of a previous modification completing.

### During Change

* **Wait for `optimizing` or `completed` before proceeding to OS steps** -- The `modifying` state means the resize is in progress at the AWS layer. Attempting `growpart` while the volume is still in `modifying` state can produce inconsistent results.
* **Use `growpart` over manual `fdisk` for live systems** -- `growpart` is designed to be safe on mounted, live partitions. Manual `fdisk` edits on a live root partition carry significantly higher risk.
* **XFS supports online expansion -- ext4 requires unmount for shrink** -- `xfs_growfs` can expand a mounted XFS filesystem without unmount. Note that XFS cannot be shrunk; ext4 requires `resize2fs`.

### Post-Change

* **Always verify with both `lsblk` and `df -hT`** -- `lsblk` confirms the block device and partition view; `df` confirms the filesystem view. Both must align before the task is complete.
* **Document the before and after state** -- Record the pre-change and post-change `df` and `lsblk` outputs for audit trail and change management tickets.
* **Log all AWS CLI commands to a runbook** -- Reproducible runbooks reduce MTTR for future incidents of the same class.

---

## Lessons Learned

1. **EBS volume expansion is only the first layer.** The 3-layer model (EBS block device, partition table, filesystem) must be understood clearly. Each layer must be expanded independently. A common production incident occurs when engineers complete the AWS-side step and close the ticket, leaving the OS still reporting the original size.

2. **`optimizing` does not mean `incomplete`.** The AWS `optimizing` state means the resize is fully done -- storage is available. AWS continues to optimize I/O performance in the background. You do not need to wait for `completed` before proceeding with OS-level steps.

3. **`growpart` is idempotent and safe.** Unlike manual partition table editing, `growpart` will return `NOCHANGE` if the partition already fills the disk, making it safe to run in automation and in Ansible/Terraform provisioning scripts.

4. **`xfs_growfs` requires the filesystem to be mounted.** Unlike `resize2fs` (ext4), `xfs_growfs` can only expand a filesystem that is currently mounted. This is actually a feature, not a limitation -- it enables fully online, zero-downtime resizing.

5. **`chmod 400` on PEM files is non-negotiable.** SSH enforces strict key file permission checks. A PEM file with permissions `644` or wider will cause SSH to reject it with `Permissions too open`. Always store and use key pairs with `400` permissions.

6. **GPT PMBR warnings are expected after EBS expansion.** These are not errors. They are a natural artifact of the block device growing outside the original GPT header boundaries. `growpart` resolves them atomically.

7. **Use tag-based queries, not hardcoded IDs, in runbooks.** Querying by `Name` tag (`datacenter-ec2`) rather than hardcoded instance IDs makes runbooks portable across environments and prevents accidental application to the wrong resource after instance replacements.

---

## Quick Reference Cheatsheet

```bash
# 1. Find instance and volume
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=<instance-name>" \
  --region <region> \
  --query "Reservations[*].Instances[*].[InstanceId,State.Name,BlockDeviceMappings[*].Ebs.VolumeId]" \
  --output table

# 2. Confirm volume details
aws ec2 describe-volumes \
  --filters "Name=attachment.instance-id,Values=<instance-id>" \
  --region <region> \
  --query "Volumes[*].[VolumeId,Size,State,VolumeType,Attachments[0].Device]" \
  --output table

# 3. Expand volume
aws ec2 modify-volume --volume-id <volume-id> --size <new-size-gb> --region <region>

# 4. Wait for optimizing state
aws ec2 describe-volumes-modifications \
  --volume-ids <volume-id> \
  --region <region> \
  --query "VolumesModifications[*].[VolumeId,ModificationState,TargetSize,Progress]" \
  --output table

# 5. SSH in
chmod 400 /path/to/keypair.pem
ssh -i /path/to/keypair.pem ec2-user@<public-ip>

# --- Inside the instance ---

# 6. Verify kernel sees new size
lsblk

# 7. Check filesystem (still old size at this point)
df -hT /

# 8. Inspect partition table (expect GPT PMBR warning -- safe to ignore)
sudo fdisk -l

# 9. Grow the partition
sudo growpart /dev/xvda 1

# 10. Confirm partition expanded
lsblk

# 11. Expand XFS filesystem (online, no unmount needed)
sudo xfs_growfs -d /

# 12. Validate
df -hT /
```

---

Region: `us-east-1`
Classification: Storage Capacity Expansion -- P2 Operational

---

*All commands are idempotent-safe unless noted. For ext4 filesystems, replace `xfs_growfs -d /` with `sudo resize2fs /dev/xvda1`.*












<img width="1031" height="727" alt="image" src="https://github.com/user-attachments/assets/8a0dbb2c-0757-4d32-a215-b063e05b8cfb" />
<img width="1037" height="601" alt="image" src="https://github.com/user-attachments/assets/b7997691-281e-4fd7-8fcc-6ef3a6172734" />


