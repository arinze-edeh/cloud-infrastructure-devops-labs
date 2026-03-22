# Azure Blob Storage Container Migration and Cleanup

> **Enterprise Cloud Operations | Azure Storage | DevOps**

![Azure](https://img.shields.io/badge/Azure-Storage-0089D6?style=flat-square&logo=microsoft-azure&logoColor=white)
![CLI](https://img.shields.io/badge/Azure%20CLI-2.67.0-blue?style=flat-square)
![Status](https://img.shields.io/badge/Status-Resolved-brightgreen?style=flat-square)
![Category](https://img.shields.io/badge/Category-Cloud%20Cleanup-orange?style=flat-square)

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Environment and Prerequisites](#environment-and-prerequisites)
- [Architecture Overview](#architecture-overview)
- [Resolution Walkthrough](#resolution-walkthrough)
  - [Phase 1: Environment Verification](#phase-1-environment-verification)
  - [Phase 2: Storage Account Discovery](#phase-2-storage-account-discovery)
  - [Phase 3: Key Retrieval and Container Inspection](#phase-3-key-retrieval-and-container-inspection)
  - [Phase 4: Blob Download to Local Host](#phase-4-blob-download-to-local-host)
  - [Phase 5: Container Deletion and Verification](#phase-5-container-deletion-and-verification)
- [Validation and Verification](#validation-and-verification)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Reference Commands](#reference-commands)

---

## Problem Statement

### Context

The Nautilus DevOps team initiated an Azure environment cleanup initiative targeting one-time-use resources provisioned during a migration project. A private blob container holding transient data was identified for decommission after its contents were safely migrated to the target compute host.

### Objectives

| # | Objective | Target Resource |
|---|-----------|-----------------|
| 1 | Copy all blob contents to the local host | `xfusion-blob-11509` to `/opt` on `azure-client` |
| 2 | Permanently delete the blob container | `xfusion-blob-11509` from `xfusionst27608` |

### Affected Resources

| Resource Type | Resource Name | Region |
|---------------|---------------|--------|
| Storage Account | `xfusionst27608` | South Central US |
| Blob Container | `xfusion-blob-11509` | Inherited from account |
| Resource Group | `kml_rg_main-ca1fc5d65be64345` | South Central US |
| Target Host | `azure-client` | On-premise |
| Target Directory | `/opt` | `azure-client` filesystem |

---

## Environment and Prerequisites

### Tool Versions

```
azure-cli    2.67.0
Python       3.10.15 (Linux)
OS           Linux (azure-client)
```

### Authentication

Authentication was performed using a **Service Principal** under the `Azure Free Labs` subscription.

```json
{
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "name": "Azure Free Labs",
  "user": {
    "type": "servicePrincipal"
  }
}
```

> **Security Note:** Never commit Service Principal credentials or storage account keys to version control. Use Azure Key Vault or environment variable injection in CI/CD pipelines.

### Required Permissions

| Permission Scope | Required Role |
|------------------|---------------|
| List storage account keys | `Storage Account Contributor` or `Owner` |
| Download blobs | `Storage Blob Data Reader` (minimum) |
| Delete blob container | `Storage Blob Data Contributor` or `Owner` |

---

## Architecture Overview

```
+--------------------+        az storage         +------------------------+
|   azure-client     |  <----  download-batch --- |  Azure Storage Account |
|   Host: /opt/      |                            |  xfusionst27608        |
+--------------------+                            |  Container:            |
                                                  |  xfusion-blob-11509    |
                                                  |  Blob: xfusion.txt     |
                                                  +------------------------+
                                                            |
                                                            v
                                                   [DELETE CONTAINER]
                                                   xfusion-blob-11509
```

---

## Resolution Walkthrough

---

### Phase 1: Environment Verification

**Objective:** Confirm the Azure CLI version and active subscription context before executing any resource operations.

#### Step 1.1 -- Verify Azure CLI Version

```bash
az --version
```

**Expected Output:**

```
azure-cli    2.67.0
Python (Linux) 3.10.15
```

#### Step 1.2 -- Confirm Active Subscription

```bash
az account show
```

**Expected Output:**

```json
{
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": {
    "name": "49adffa8-1256-4c01-a8d8-dac8dc36904f",
    "type": "servicePrincipal"
  }
}
```

#### Step 1.3 -- List All Subscriptions (Audit Trail)

```bash
az account list --output table
```

> **Screenshot**

<img width="1037" height="838" alt="image" src="https://github.com/user-attachments/assets/a4dfd370-301c-47aa-aea5-7485a61b3888" />

> `az account list table showing single subscription "Azure Free Labs" as default`

---

### Phase 2: Storage Account Discovery

**Objective:** Locate the target storage account and resolve its parent resource group dynamically.

#### Step 2.1 -- Resolve Resource Group from Storage Account Name

```bash
az storage account list \
  --query "[?name=='xfusionst27608'].resourceGroup" \
  --output tsv
```

**Output:**

```
kml_rg_main-ca1fc5d65be64345
```

> **Screenshot**

<img width="1033" height="537" alt="image" src="https://github.com/user-attachments/assets/8c21fdc0-5c5f-4c9a-99a1-fcdd6273121c" />

> `az storage account list query output returning resource group "kml_rg_main-ca1fc5d65be64345"`

#### Step 2.2 -- Export Resource Group to Shell Variable

```bash
RG=kml_rg_main-ca1fc5d65be64345
echo $RG
```

**Output:**

```
kml_rg_main-ca1fc5d65be64345
```

> **Screenshot**

<img width="1029" height="634" alt="image" src="https://github.com/user-attachments/assets/8e6d165a-d300-497e-8c26-e7a45037dfd8" />

> `Shell variable RG set and echoed to confirm correct value`

---

### Phase 3: Key Retrieval and Container Inspection

**Objective:** Securely retrieve the storage account access key and validate the target container existence and blob inventory.

#### Step 3.1 -- Retrieve Storage Account Access Key

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name xfusionst27608 \
  --resource-group $RG \
  --query "[0].value" \
  --output tsv)

echo $STORAGE_KEY
```

**Output:**

```
/OYRMyRWEwir8X5kMFpD/rf5s7cfBzn...==
```

> **Screenshotr**

<img width="1035" height="497" alt="image" src="https://github.com/user-attachments/assets/98bf154f-3a1b-422c-b73a-3c564326b1d8" />

> `STORAGE_KEY variable populated with base64-encoded storage key value`

> **Security Advisory:** Avoid echoing storage keys in production environments or shared terminal sessions. Pipe directly to secure variables or use Azure Key Vault references.

#### Step 3.2 -- Inspect Target Container Metadata

```bash
az storage container show \
  --name xfusion-blob-11509 \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY
```

**Key Properties Validated:**

| Property | Value |
|----------|-------|
| Container Name | `xfusion-blob-11509` |
| Public Access | `null` (private) |
| Lease State | `available` |
| Lease Status | `unlocked` |
| Has Immutability Policy | `false` |
| Has Legal Hold | `false` |

> **Screenshot**

<img width="1029" height="844" alt="image" src="https://github.com/user-attachments/assets/1c39ff5b-0bc5-4774-bc53-5ba7ec3d4529" />

> `az storage container show output confirming container is unlocked, private, and has no legal hold`

#### Step 3.3 -- List Blobs in Container

```bash
az storage blob list \
  --container-name xfusion-blob-11509 \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY \
  --output table
```

**Output:**

```
Name         Blob Type    Blob Tier    Length    Content Type    Last Modified
-----------  -----------  -----------  --------  --------------  -------------------------
xfusion.txt  BlockBlob    Hot          33        text/plain      2026-03-22T04:05:58+00:00
```

> **Screenshot**

<img width="1031" height="748" alt="image" src="https://github.com/user-attachments/assets/804a4b44-1545-4d58-a11a-8e736f837057" />

> `Blob list table output showing single blob "xfusion.txt" as BlockBlob on Hot tier`

#### Step 3.4 -- Count Total Blobs (Pre-Download Baseline)

```bash
az storage blob list \
  --container-name xfusion-blob-11509 \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY \
  --query "length(@)" \
  --output tsv
```

**Output:**

```
1
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Blob count query returning "1" confirming exactly one blob exists before download]`

---

### Phase 4: Blob Download to Local Host

**Objective:** Download all blobs from the container to the `/opt` directory on the `azure-client` host and verify successful transfer.

#### Step 4.1 -- Confirm Target Directory Exists and Is Writable

```bash
ls -ld /opt
touch /opt/.write_test && echo "Write access confirmed" && rm /opt/.write_test
```

**Output:**

```
drwxr-xr-x 1 root root 4096 Mar 22 04:05 /opt
Write access confirmed
```

> **Screenshot Placeholder**
> `[SCREENSHOT: ls -ld /opt showing directory permissions and write test confirming access]`

#### Step 4.2 -- Execute Batch Download from Container to /opt

```bash
az storage blob download-batch \
  --destination /opt \
  --source xfusion-blob-11509 \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY
```

**Output:**

```
Finished[#############################################################]  100.0000%
[
  "xfusion.txt"
]
```

> **Screenshot Placeholder**
> `[SCREENSHOT: az storage blob download-batch progress bar completing at 100% with "xfusion.txt" listed]`

#### Step 4.3 -- Verify Downloaded File in /opt

```bash
ls -lah /opt/
```

**Output (post-download):**

```
total 28K
drwxr-xr-x 1 root root 4.0K Mar 22 04:25 .
dr-xr-xr-x 1 root root 4.0K Mar 22 04:06 ..
-rw-r--r-- 1 root root  445 Mar 22 04:05 creds.json
drwxr-xr-x 1 root root 4.0K Mar 22 03:02 .init
-rw-r--r-- 1 root root 2.7K Mar 22 04:05 showcreds
-rw-r--r-- 1 root root   33 Mar 22 04:25 xfusion.txt
```

> **Screenshot Placeholder**
> `[SCREENSHOT: ls -lah /opt showing xfusion.txt downloaded at 04:25 with 33 bytes matching blob size]`

#### Step 4.4 -- Verify File Count in /opt

```bash
ls /opt/ | wc -l
```

**Output:**

```
3
```

> **Screenshot Placeholder**
> `[SCREENSHOT: ls /opt | wc -l output confirming 3 visible files in /opt after download]`

#### Step 4.5 -- Verify Downloaded File in /opt

Confirm the blob landed correctly in the destination directory:

```bash
ls -lah /opt/
```

**Output (post-download):**

```
total 28K
drwxr-xr-x 1 root root 4.0K Mar 22 04:25 .
dr-xr-xr-x 1 root root 4.0K Mar 22 04:06 ..
-rw-r--r-- 1 root root  445 Mar 22 04:05 creds.json
drwxr-xr-x 1 root root 4.0K Mar 22 03:02 .init
-rw-r--r-- 1 root root 2.7K Mar 22 04:05 showcreds
-rw-r--r-- 1 root root   33 Mar 22 04:25 xfusion.txt
```

> **Screenshot Placeholder**
> `[SCREENSHOT: ls -lah /opt showing xfusion.txt downloaded at 04:25 with 33 bytes matching blob size]`

---

### Phase 5: Container Deletion and Verification

**Objective:** Permanently delete the blob container from the storage account and perform multi-step verification to confirm complete removal.

#### Step 5.1 -- Delete the Blob Container

```bash
az storage container delete \
  --name xfusion-blob-11509 \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY
```

**Output:**

```json
{
  "deleted": true
}
```

> `az storage container delete returning {"deleted": true} confirming deletion accepted by Azure`

#### Step 5.2 -- Confirm Container No Longer Exists

```bash
az storage container exists \
  --name xfusion-blob-11509 \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY
```

**Output:**

```json
{
  "exists": false
}
```

> ` az storage container exists returning {"exists": false} confirming container is permanently gone`

#### Step 5.3 -- List All Remaining Containers (Final State Audit)

```bash
az storage container list \
  --account-name xfusionst27608 \
  --account-key $STORAGE_KEY \
  --output table
```

**Output:**

```
(empty)
```

> **Screenshot**

<img width="1011" height="405" alt="image" src="https://github.com/user-attachments/assets/b3450cf5-b532-40c4-a708-70c77e6a581e" />


> `az storage container list returning empty table confirming no containers remain in xfusionst27608`

#### Step 5.4 -- Final File Integrity Verification on Local Host

With the container confirmed deleted, perform a final check on the locally downloaded file to close the end-to-end audit loop.

Inspect file metadata:

```bash
ls -lah /opt/xfusion.txt
```

**Output:**

```
-rw-r-- r -- 1 root root 33 Mar 22 04:25 /opt/xfusion.txt
```

Read and verify file content:

```bash
cat /opt/xfusion.txt
```

**Output:**

```
Welcome to KKE Azure Cloud Labs!
```

> **Screenshot**

<img width="1032" height="580" alt="image" src="https://github.com/user-attachments/assets/febfeca1-ef93-42fb-a656-82f196ea6829" />

> `ls -lah /opt/xfusion.txt showing 33-byte file (root-owned, Mar 22 04:25) followed immediately by cat /opt/xfusion.txt output displaying "Welcome to KKE Azure Cloud Labs!"`

> **Integrity Check:** File size on disk (33 bytes) matches the blob `Length` field (33) from the pre-download inventory. File permissions are `rw-r--r--` (root-owned, world-readable). The source container is deleted and the data is confirmed safe and intact on the local host. Migration and cleanup complete.

---

## Validation and Verification

### Pre and Post State Comparison

| Checkpoint | Pre-Operation State | Post-Operation State |
|------------|---------------------|----------------------|
| Container `xfusion-blob-11509` | Exists | Deleted |
| Blob `xfusion.txt` | In Azure container | Copied to `/opt/xfusion.txt` |
| File size integrity | 33 bytes (Azure) | 33 bytes (local) |
| Container count in storage account | 1 | 0 |
| `/opt` write access | Confirmed | Retained |

### End-to-End Verification Checklist

- [x] Azure CLI authenticated with correct Service Principal
- [x] Target subscription `Azure Free Labs` confirmed active
- [x] Resource group resolved dynamically from storage account query
- [x] Storage account key retrieved and stored in shell variable
- [x] Container metadata inspected: no legal hold, no immutability policy
- [x] Blob inventory confirmed (1 blob, 33 bytes)
- [x] Write access to `/opt` verified prior to download
- [x] `download-batch` completed at 100% with correct blob listed
- [x] File `xfusion.txt` present in `/opt` with matching size
- [x] File content readable and intact
- [x] Container deletion returned `{"deleted": true}`
- [x] Post-delete existence check returned `{"exists": false}`
- [x] Container list confirmed empty storage account

---

## Best Practices

### Credential and Key Management

* **Never hardcode storage keys** in scripts, Makefiles, or pipeline YAML. Use environment variables injected at runtime or reference secrets via Azure Key Vault.
* **Rotate storage account keys** on a regular schedule. Azure provides two keys (key1, key2) to enable zero-downtime rotation.
* **Use SAS tokens** for scoped, time-limited access instead of full account keys wherever possible. Apply the principle of least privilege.
* **Audit key usage** via Azure Monitor storage diagnostics to detect unauthorized access.

### Pre-Deletion Checklist

* Always **list and count blobs** before deletion to establish a baseline (`--query "length(@)"`).
* Validate that no **immutability policies or legal holds** are active on the container (`az storage container show`).
* Confirm the container has **no soft-delete retention window** that could cause unexpected charges post-deletion.
* Cross-check blob sizes between source and destination to confirm **transfer integrity** before deleting the source.

### Shell Scripting Standards

* Export repeated values (resource group, storage key) to **named shell variables** at the top of your session to avoid repetition and typos.
* Use `--output tsv` for programmatic capture and `--output table` for human-readable audit trails.
* Use `--query` JMESPath filters to extract only the needed fields rather than parsing full JSON output in downstream tools.
* Always run `echo $VARIABLE` immediately after assignment to confirm the correct value was captured.

### Disaster Recovery Posture

* Before deleting any cloud resource, confirm that **blob soft-delete is disabled** if you want immediate, permanent removal, or that it is enabled if you need a recovery window.
* Consider enabling **Azure Storage lifecycle management policies** to automate future cleanup rather than performing manual deletions.
* Tag one-time-use resources at creation time (`az storage container create --metadata purpose=migration`) to make future cleanup queries trivial.

### Access Control

* Prefer **RBAC role assignments** (`Storage Blob Data Contributor`) over shared key authentication for production workloads. Shared keys grant full account-level access with no audit trail granularity.
* Log all CLI sessions performing destructive operations (delete, purge) to an immutable audit store.

---

## Lessons Learned

### 1. Dynamic Resource Group Resolution Prevents Hardcoding Errors

Rather than hardcoding the resource group name, the `--query` filter on `az storage account list` resolved it dynamically. This pattern is essential in multi-environment pipelines where resource group names differ between dev, staging, and production.

```bash
# Portable, environment-agnostic pattern
RG=$(az storage account list \
  --query "[?name=='STORAGE_ACCOUNT_NAME'].resourceGroup" \
  --output tsv)
```

### 2. Pre-Download Write Access Verification Prevents Silent Failures

Testing write access to `/opt` before initiating the download (`touch /opt/.write_test`) confirmed the operation would succeed. Without this step, a permissions failure mid-batch could leave a partial download with no clear error signal.

### 3. Blob Count and Size Verification Closes the Data Integrity Loop

Capturing `az storage blob list --query "length(@)"` before the download and then comparing file size post-download (`ls -lah /opt/xfusion.txt`) against the blob `Length` field provided a deterministic integrity check. File content inspection (`cat`) provided a final human-readable confirmation.

### 4. Multi-Step Container Deletion Verification Is Non-Negotiable

A single `{"deleted": true}` response is not sufficient for production runbooks. Combining:
1. `az storage container delete` (action)
2. `az storage container exists` (boolean check)
3. `az storage container list` (full account audit)

...provides a three-layer confirmation that the resource is truly gone and no other containers were inadvertently affected.

### 5. Shell Variable Scoping Improves Repeatability and Reduces Typos

Setting `RG` and `STORAGE_KEY` as shell variables at the start of the session eliminated the risk of typographic errors in repeated resource identifiers across seven distinct `az` commands. In production automation, these would be exported or sourced from a secrets manager.

### 6. `download-batch` Is Preferred Over Per-Blob Downloads for Scalability

`az storage blob download-batch` handles multi-blob containers atomically with built-in progress reporting. Even though only one blob was present here, using `download-batch` over `az storage blob download` establishes a pattern that scales without script modification as blob counts grow.

### 7. Validate the Azure CLI Version and Subscription Context at Session Start

Confirming `az --version` and `az account show` at session start is a low-cost step that prevents an entire runbook from executing against the wrong subscription or a stale CLI version with known bugs.

---

## Reference Commands

### Quick Reference Sheet

```bash
# Resolve resource group from storage account name
az storage account list \
  --query "[?name=='<STORAGE_ACCOUNT>'].resourceGroup" \
  --output tsv

# Retrieve primary storage account key
az storage account keys list \
  --account-name <STORAGE_ACCOUNT> \
  --resource-group <RESOURCE_GROUP> \
  --query "[0].value" --output tsv

# Inspect container metadata
az storage container show \
  --name <CONTAINER> \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY>

# List blobs with count
az storage blob list \
  --container-name <CONTAINER> \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY> --output table

az storage blob list \
  --container-name <CONTAINER> \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY> --query "length(@)" --output tsv

# Batch download all blobs to a local directory
az storage blob download-batch \
  --destination <LOCAL_DIR> \
  --source <CONTAINER> \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY>

# Delete container
az storage container delete \
  --name <CONTAINER> \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY>

# Verify container is gone
az storage container exists \
  --name <CONTAINER> \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY>

# List all remaining containers
az storage container list \
  --account-name <STORAGE_ACCOUNT> \
  --account-key <KEY> --output table
```

---

## Document Metadata

| Field | Value |
|-------|-------|
| Author | Nautilus DevOps Team |
| Runbook Type | Cloud Cleanup / Migration |
| Platform | Microsoft Azure |
| CLI Version | `azure-cli 2.67.0` |
| Execution Host | `azure-client` |
| Execution Date | 2026-03-22 |
| Subscription | Azure Free Labs |
| Storage Account | `xfusionst27608` |
| Container Decommissioned | `xfusion-blob-11509` |
| Status | Completed and Verified |

---









<img width="1031" height="386" alt="image" src="https://github.com/user-attachments/assets/85f5d510-77fc-4646-8f9c-691fd3404ea1" />
<img width="1032" height="672" alt="image" src="https://github.com/user-attachments/assets/37655035-ab1f-4470-9141-6bbccfe0ca61" />
<img width="1036" height="665" alt="image" src="https://github.com/user-attachments/assets/f5dcc5d0-538e-4ede-bd61-eaa218cffcee" />

