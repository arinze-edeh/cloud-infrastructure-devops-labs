# Azure SQL Database Provisioning, Backup, and Recovery Pipeline
  
> Scope: Azure SQL Database lifecycle management, blob-based BACPAC export, and verified local recovery

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Environment Variables](#environment-variables)
- [Task 1: Provision the Azure SQL Server and Database](#task-1-provision-the-azure-sql-server-and-database)
- [Task 2: Create the Azure Storage Account and Blob Container](#task-2-create-the-azure-storage-account-and-blob-container)
- [Task 3: Export the Database Backup to Blob Storage](#task-3-export-the-database-backup-to-blob-storage)
- [Task 4: Download and Verify the Backup Locally](#task-4-download-and-verify-the-backup-locally)
- [Verification Checklist](#verification-checklist)
- [Errors Encountered](#errors-encountered)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Reference](#reference)

---

## Project Overview

The Nautilus DevOps team is executing an incremental, phase-based migration of on-premises infrastructure to Microsoft Azure. This runbook documents the provisioning of an Azure SQL Database, the configuration of supporting blob storage, the execution of a full BACPAC database export, and the download and local verification of that backup asset.

This document is intended for Site Reliability Engineers, Cloud Infrastructure Engineers, and DevOps practitioners responsible for data tier migrations, backup management, and disaster recovery readiness on the Azure platform.

**Objective:** Provision a publicly accessible Azure SQL Database, export a verified BACPAC backup to Azure Blob Storage, and confirm successful local download of the backup artifact.

---

## Architecture Summary

```
+-----------------------------+
|   Azure Resource Group      |
|   kml_rg_main-121b3e04c4.. |
|                             |
|  +----------------------+   |
|  | Azure SQL Server     |   |
|  | xfusion-server-3853  |   |
|  |  Region: West US     |   |
|  |                      |   |
|  |  +----------------+  |   |
|  |  | SQL Database   |  |   |
|  |  | xfusion-sqldb  |  |   |
|  |  | Tier: Basic    |  |   |
|  |  | Size: 2 GiB    |  |   |
|  |  +----------------+  |   |
|  +----------------------+   |
|                             |
|  +----------------------+   |
|  | Storage Account      |   |
|  | xfusionst20759       |   |
|  |  SKU: Standard_LRS   |   |
|  |                      |   |
|  |  +----------------+  |   |
|  |  | Blob Container |  |   |
|  |  | xfusion-       |  |   |
|  |  | container-22348|  |   |
|  |  | .bacpac file   |  |   |
|  |  +----------------+  |   |
|  +----------------------+   |
+-----------------------------+
          |
          | az storage blob download
          v
+-----------------------------+
|   azure-client host         |
|   /opt/xfusion-db-backup    |
|   .bacpac  (2.8K verified)  |
+-----------------------------+
```

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Installed and authenticated (`az login`) |
| Active Azure Subscription | Subscription ID: `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Existing Resource Group | `kml_rg_main-121b3e04c4694db4` in `eastus` |
| Permissions | Contributor or Owner role on the resource group |
| Target Region | `westus` (all new resources provisioned here) |
| Local Host | `azure-client` with write access to `/opt/` |

---

## Environment Variables

All variables were defined at session start. Every subsequent command referenced these variables to maintain consistency and eliminate hardcoded values.

```bash
RESOURCE_GROUP=$(az group list --query "[0].name" --output tsv)
LOCATION="westus"
SQL_SERVER="xfusion-server-3853"
SQL_DB="xfusion-sqldb"
SQL_ADMIN="xfusion-admin"
SQL_PASS="N@utilus#Secure2024!"
STORAGE_ACCOUNT="xfusionst20759"
CONTAINER="xfusion-container-22348"
BACKUP_FILE="xfusion-db-backup"
```

**Verified output:**

```
RG       : kml_rg_main-121b3e04c4694db4
Location : westus
Server   : xfusion-server-3853
DB       : xfusion-sqldb
Admin    : xfusion-admin
Storage  : xfusionst20759
Container: xfusion-container-22348
Backup   : xfusion-db-backup
```

> **Screenshot**

<img width="1227" height="706" alt="image" src="https://github.com/user-attachments/assets/429d2818-492f-45f7-93b5-56a1f7e50413" />

> `Terminal output showing all environment variable echo statements confirmed`

---

## Task 1: Provision the Azure SQL Server and Database

### Step 1.1 -- Confirm the Resource Group

Before provisioning, the existing resource group was confirmed.

```bash
az group list --output table
```

**Output:**

```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-121b3e04c4694db4  eastus      Succeeded
```

> **Screenshot**

<img width="1232" height="509" alt="image" src="https://github.com/user-attachments/assets/93694deb-0e6a-40c6-ab6c-e242817fea7a" />

> `az group list --output table showing kml_rg_main-121b3e04c4694db4 in Succeeded state`

---

### Step 1.2 -- Create the Azure SQL Server

The SQL logical server was provisioned in `westus` with TLS 1.2 enforced and public network access enabled to satisfy the publicly accessible requirement.

```bash
az sql server create \
  --name "$SQL_SERVER" \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --admin-user "$SQL_ADMIN" \
  --admin-password "$SQL_PASS"
```

**Key response fields:**

```json
{
  "fullyQualifiedDomainName": "xfusion-server-3853.database.windows.net",
  "location": "westus",
  "minimalTlsVersion": "1.2",
  "name": "xfusion-server-3853",
  "publicNetworkAccess": "Enabled",
  "state": "Ready",
  "version": "12.0"
}
```

> **Screenshot**

<img width="1232" height="870" alt="image" src="https://github.com/user-attachments/assets/c3101ee3-b131-45e0-9449-73dd67e13405" />

> `Full JSON output from az sql server create confirming state: Ready and publicNetworkAccess: Enabled`

---

### Step 1.3 -- Configure Firewall Rule for Azure Services

A firewall rule was created with both start and end IP set to `0.0.0.0`. This is the Azure-native convention that permits all Azure-originated service traffic (such as the export job) while blocking external internet access.

```bash
az sql server firewall-rule create \
  --resource-group "$RESOURCE_GROUP" \
  --server "$SQL_SERVER" \
  --name "AllowAllAzureIPs" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

**Output:**

```json
{
  "endIpAddress": "0.0.0.0",
  "name": "AllowAllAzureIPs",
  "startIpAddress": "0.0.0.0"
}
```

> **Screenshot**

<img width="1232" height="865" alt="image" src="https://github.com/user-attachments/assets/fe6f7219-83b0-4bb2-a65a-5d4ade1384df" />

> `Firewall rule creation output showing AllowAllAzureIPs with 0.0.0.0/0.0.0.0 range`

---

### Step 1.4 -- Create the Azure SQL Database

The database was provisioned on the Basic tier with 5 DTUs, a 2 GiB maximum size, and locally redundant backup storage.

```bash
az sql db create \
  --resource-group "$RESOURCE_GROUP" \
  --server "$SQL_SERVER" \
  --name "$SQL_DB" \
  --edition "Basic" \
  --capacity 5 \
  --max-size 2147483648 \
  --backup-storage-redundancy "Local"
```

**Key response fields:**

```json
{
  "name": "xfusion-sqldb",
  "edition": "Basic",
  "currentBackupStorageRedundancy": "Local",
  "maxSizeBytes": 2147483648,
  "status": "Online",
  "location": "westus"
}
```

> **Screenshots**

<img width="1236" height="857" alt="image" src="https://github.com/user-attachments/assets/e8422294-b1ed-447f-b6d3-3058eaaa359e" />
<img width="1233" height="866" alt="image" src="https://github.com/user-attachments/assets/a3c946aa-41b6-4156-9818-3669736cc462" />
<img width="1232" height="856" alt="image" src="https://github.com/user-attachments/assets/8b3fec08-5585-41e6-9121-aa6d6a850dbc" />

> `az sql db create full JSON output with status: Online and edition: Basic confirmed`

---

### Step 1.5 -- Verify Database Status

The database status was explicitly queried to confirm readiness before proceeding.

```bash
az sql db show \
  --resource-group "$RESOURCE_GROUP" \
  --server "$SQL_SERVER" \
  --name "$SQL_DB" \
  --query "status" \
  --output tsv
```

**Output:**

```
Online
```

> **Screenshot**

<img width="1231" height="392" alt="image" src="https://github.com/user-attachments/assets/fe52c907-7939-48b7-9de6-b5fe53014589" />

> `Terminal showing "Online" status returned from az sql db show query`

---

## Task 2: Create the Azure Storage Account and Blob Container

### Step 2.1 -- Create the Storage Account

A StorageV2 account with Standard LRS redundancy was provisioned in `westus` to host the backup container.

```bash
az storage account create \
  --name "$STORAGE_ACCOUNT" \
  --resource-group "$RESOURCE_GROUP" \
  --location "$LOCATION" \
  --sku "Standard_LRS" \
  --kind "StorageV2"
```

**Key response fields:**

```json
{
  "name": "xfusionst20759",
  "kind": "StorageV2",
  "location": "westus",
  "provisioningState": "Succeeded",
  "primaryEndpoints": {
    "blob": "https://xfusionst20759.blob.core.windows.net/"
  },
  "sku": {
    "name": "Standard_LRS",
    "tier": "Standard"
  },
  "statusOfPrimary": "available"
}
```

> **Screenshots**

<img width="1232" height="854" alt="image" src="https://github.com/user-attachments/assets/caac2a49-45b3-4854-9393-4dbfdbad4602" />
<img width="1234" height="860" alt="image" src="https://github.com/user-attachments/assets/2cd265b6-d987-4350-9bf3-76e6702a3e45" />
<img width="1232" height="862" alt="image" src="https://github.com/user-attachments/assets/0f9bcb00-d8a9-4168-b6be-9b32af29aa70" />

> `az storage account create output with provisioningState: Succeeded and blob endpoint visible`

---

### Step 2.2 -- Retrieve the Storage Account Access Key

The first account key was retrieved and stored in a variable. The key was masked in output to avoid credential exposure in logs.

```bash
STORAGE_KEY=$(az storage account keys list \
  --resource-group "$RESOURCE_GROUP" \
  --account-name "$STORAGE_ACCOUNT" \
  --query "[0].value" \
  --output tsv)
echo "Key: ${STORAGE_KEY:0:10}..."
```

**Output:**

```
Key: MpSsw8pPXB...
```

> **Screenshot**

<img width="1227" height="590" alt="image" src="https://github.com/user-attachments/assets/92bb2ef3-ae79-4491-bbda-9f16ed28eb04" />

> `Terminal output showing masked storage key prefix confirming successful key retrieval`

---

### Step 2.3 -- Create the Blob Container

The blob container was created under the storage account using the retrieved access key.

```bash
az storage container create \
  --name "$CONTAINER" \
  --account-name "$STORAGE_ACCOUNT" \
  --account-key "$STORAGE_KEY"
```

**Output:**

```json
{
  "created": true
}
```

> **Screenshot**

<img width="1225" height="494" alt="image" src="https://github.com/user-attachments/assets/db0eaf97-c677-4eb8-95ae-750959d2c7dc" />

> `az storage container create output showing created: true`

---

### Step 2.4 -- Verify Container Existence

The container list was queried to confirm successful creation before proceeding to the backup step.

```bash
az storage container list \
  --account-name "$STORAGE_ACCOUNT" \
  --account-key "$STORAGE_KEY" \
  --output table
```

**Output:**

```
Name                     Lease Status    Last Modified
-----------------------  --------------  -------------------------
xfusion-container-22348                  2026-03-28T05:16:40+00:00
```

> **Screenshot**

<img width="1230" height="499" alt="image" src="https://github.com/user-attachments/assets/35c69824-4a70-4bab-984d-b86c2945280f" />

> `az storage container list output confirming xfusion-container-22348 exists`

---

## Task 3: Export the Database Backup to Blob Storage

### Step 3.1 -- Generate a SAS Token

A Shared Access Signature (SAS) token was generated for the storage account, scoped to blob service operations with full read/write/delete permissions and a 3-hour expiry window.

```bash
END_DATE=$(date -u -d "+3 hours" '+%Y-%m-%dT%H:%MZ')

SAS_TOKEN=$(az storage account generate-sas \
  --account-name "$STORAGE_ACCOUNT" \
  --account-key "$STORAGE_KEY" \
  --expiry "$END_DATE" \
  --permissions "rwdlacup" \
  --resource-types "sco" \
  --services "b" \
  --output tsv)
echo "SAS OK: ${SAS_TOKEN:0:20}..."
```

**Output:**

```
SAS OK: se=2026-03-28T08%3A1...
```

> **Screenshot**

<img width="1233" height="739" alt="image" src="https://github.com/user-attachments/assets/49ebef8a-067b-404b-b701-9aae5f6a4146" />

> `Terminal showing SAS token generation with masked prefix confirming success and expiry date`

---

### Step 3.2 -- Construct the Backup Target URI

The full BACPAC blob URI was assembled from the storage account name, container name, and backup file name.

```bash
STORAGE_URI="https://${STORAGE_ACCOUNT}.blob.core.windows.net/${CONTAINER}/${BACKUP_FILE}.bacpac"
echo "URI: $STORAGE_URI"
```

**Output:**

```
URI: https://xfusionst20759.blob.core.windows.net/xfusion-container-22348/xfusion-db-backup.bacpac
```

> **Screenshot**

<img width="1230" height="669" alt="image" src="https://github.com/user-attachments/assets/8448a148-cd53-402b-b0f9-c872d7a56f9e" />

> `Terminal showing the constructed STORAGE_URI echoed correctly`

---

### Step 3.3 -- Export the Database as BACPAC

The database was exported to the target blob URI using the SAS token as the storage credential. The Azure SQL export service orchestrated the operation server-side.

```bash
az sql db export \
  --resource-group "$RESOURCE_GROUP" \
  --server "$SQL_SERVER" \
  --name "$SQL_DB" \
  --admin-user "$SQL_ADMIN" \
  --admin-password "$SQL_PASS" \
  --storage-key "$SAS_TOKEN" \
  --storage-key-type "SharedAccessKey" \
  --storage-uri "$STORAGE_URI"
```

**Output:**

```json
{
  "blobUri": "https://xfusionst20759.blob.core.windows.net/xfusion-container-22348/xfusion-db-backup.bacpac",
  "databaseName": "xfusion-sqldb",
  "errorMessage": null,
  "lastModifiedTime": "3/28/2026 5:23:22 AM",
  "queuedTime": "3/28/2026 5:19:14 AM",
  "requestType": "ExportDatabase",
  "serverName": "xfusion-server-3853",
  "status": "Completed"
}
```

> **Screenshot**

<img width="1231" height="836" alt="image" src="https://github.com/user-attachments/assets/c6b9213a-2dd6-4bb6-8e67-0f3204cd42c9" />

> `az sql db export JSON response showing status: Completed and errorMessage: null`

---

### Step 3.4 -- Confirm BACPAC Blob Presence

The blob was listed in the container to confirm the export artifact was written successfully.

```bash
az storage blob list \
  --container-name "$CONTAINER" \
  --account-name "$STORAGE_ACCOUNT" \
  --account-key "$STORAGE_KEY" \
  --output table
```

**Output:**

```
Name                      Blob Type    Blob Tier    Length    Content Type              Last Modified
------------------------  -----------  -----------  --------  ------------------------  -------------------------
xfusion-db-backup.bacpac  BlockBlob    Hot          2771      application/octet-stream  2026-03-28T05:23:04+00:00
```

> **Screenshot**

<img width="1228" height="769" alt="image" src="https://github.com/user-attachments/assets/17273094-b117-42a3-9a73-11b830a3d067" />

> `az storage blob list table output showing xfusion-db-backup.bacpac with Length 2771 in Hot tier`

---

## Task 4: Download and Verify the Backup Locally

### Step 4.1 -- Download the BACPAC to the Local Host

The BACPAC was downloaded from the blob container to `/opt/` on the `azure-client` host using the account key for authentication.

```bash
az storage blob download \
  --container-name "$CONTAINER" \
  --name "${BACKUP_FILE}.bacpac" \
  --account-name "$STORAGE_ACCOUNT" \
  --account-key "$STORAGE_KEY" \
  --file "/opt/${BACKUP_FILE}.bacpac"
```

**Output (abbreviated):**

```
Finished[#############################################################]  100.0000%
{
  "name": "xfusion-db-backup.bacpac",
  "properties": {
    "contentLength": 2771,
    "serverEncrypted": true,
    "blobType": "BlockBlob"
  }
}
```

> **Screenshots**

<img width="1227" height="858" alt="image" src="https://github.com/user-attachments/assets/5c9121fb-074e-4f06-9be3-2689858d4b5f" />
<img width="1230" height="859" alt="image" src="https://github.com/user-attachments/assets/1083e52a-7a8b-4f8a-902e-65f916b4903c" />

> `az storage blob download terminal output showing 100% completion progress bar and JSON response`

---

### Step 4.2 -- Verify the Local File

The downloaded file was verified for presence and size using `ls`.

```bash
ls -lh /opt/${BACKUP_FILE}.bacpac
```

**Output:**

```
-rw-r--r-- 1 root root 2.8K Mar 28 05:24 /opt/xfusion-db-backup.bacpac
```

> **Screenshot**

<img width="1231" height="860" alt="image" src="https://github.com/user-attachments/assets/6eb304ca-0616-4d55-b781-2579952341d9" />

> `ls -lh output showing /opt/xfusion-db-backup.bacpac at 2.8K owned by root`

---

### Step 4.3 -- Final End-to-End Verification

A combined verification command was executed to confirm all three acceptance criteria simultaneously: database online, backup blob present, local file accessible.

```bash
az sql db show --resource-group "$RESOURCE_GROUP" --server "$SQL_SERVER" --name "$SQL_DB" --query "status" --output tsv

az storage blob list --container-name "$CONTAINER" --account-name "$STORAGE_ACCOUNT" --account-key "$STORAGE_KEY" --output table

ls -lh /opt/${BACKUP_FILE}.bacpac
```

**Composite Output:**

```
Online
Name                      Blob Type    Blob Tier    Length    Content Type              Last Modified              Snapshot
------------------------  -----------  -----------  --------  ------------------------  -------------------------  ----------
xfusion-db-backup.bacpac  BlockBlob    Hot          2771      application/octet-stream  2026-03-28T05:23:04+00:00
-rw-r--r-- 1 root root 2.8K Mar 28 05:24 /opt/xfusion-db-backup.bacpac
```

> **Screenshot**

<img width="1231" height="860" alt="image" src="https://github.com/user-attachments/assets/6eb304ca-0616-4d55-b781-2579952341d9" />

> `Combined final verification output showing Online status, blob listing, and local file confirmed in a single terminal view`

---

## Verification Checklist

| # | Acceptance Criterion | Status |
|---|---|---|
| 1 | SQL Database `xfusion-sqldb` is in `Online` state | PASSED |
| 2 | BACPAC blob `xfusion-db-backup.bacpac` exists in `xfusion-container-22348` | PASSED |
| 3 | Backup file successfully downloaded to `/opt/xfusion-db-backup.bacpac` on `azure-client` | PASSED |
| 4 | Export operation completed with `errorMessage: null` | PASSED |
| 5 | File integrity confirmed at 2771 bytes (2.8K) consistent between blob and local | PASSED |

---

## Errors Encountered

No errors were returned during command execution. All Azure CLI operations completed with expected output and no non-zero exit codes were observed throughout the session.

**Notable observation:** The export operation was queued at `05:19:14 AM` and completed at `05:23:22 AM`, a total processing time of approximately 4 minutes. This is normal behavior for the Azure SQL BACPAC export service even for small databases, as the export is executed by an Azure-managed import/export service agent, not inline.

The `errorMessage: null` field in the export response was the critical indicator of clean completion and should always be checked explicitly before proceeding to download.

---

## Best Practices

### Identity and Credential Management

* **Never commit credentials to version control.** The admin password and storage keys in this runbook are injected at runtime via shell variables and should be stored in Azure Key Vault or a secrets manager in production pipelines.
* **Rotate storage account keys after use.** SAS tokens were scoped to a 3-hour window, which is appropriate for one-time export operations. Keys used for ongoing access should be rotated regularly.
* **Use Managed Identity for production pipelines.** Replace account-key-based authentication with Azure Managed Identity wherever possible to eliminate key lifecycle overhead.

### SAS Token Scoping

* Always generate SAS tokens with the minimum required permissions. The `rwdlacup` permission set used here is broader than strictly necessary for a single export. A production SAS for this operation should use `w` (write) and `c` (create) only.
* Always set a short expiry. The 3-hour window is acceptable for interactive use. Automated pipelines should use expiry windows no longer than the expected operation duration plus a buffer.

### Storage Design

* **Use geo-redundant storage (GRS or ZRS) for production backups.** `Standard_LRS` is appropriate for lab and dev environments but does not protect against zone or region failures.
* **Enable blob versioning and soft delete** on backup containers to protect against accidental deletion or overwrite of backup artifacts.
* **Name backup files with timestamps** rather than static names (e.g., `xfusion-db-backup-20260328T0519Z.bacpac`) to support retention of multiple restore points.

### SQL Database Configuration

* **Enforce Minimum TLS Version.** TLS 1.2 was enforced by default on the created server, which is the expected minimum. Validate this on existing servers that may have been created with older defaults.
* **Firewall rule `0.0.0.0` to `0.0.0.0`** is the Azure-documented pattern for allowing Azure services. It does not permit arbitrary internet access but should be documented in your security posture.
* **Prefer Private Endpoints** over public network access for production workloads to remove the SQL server from the public internet entirely.

### Verification Discipline

* Always explicitly verify each provisioned resource before depending on it in subsequent steps. The pattern used here (verify database online before attempting export, verify blob before downloading) reflects sound operational discipline.
* Confirm `errorMessage: null` on the export response. A `Completed` status with a non-null `errorMessage` can indicate a partial or corrupted export in edge cases.

---

## Lessons Learned

**1. Export operations are asynchronous by nature.**
The `az sql db export` command blocks until the operation completes, but internally the export is queued to the Azure Import/Export service. For databases of significant size, this can take tens of minutes. Build timeout handling and status polling into any automated pipeline wrapping this command.

**2. SAS tokens must be generated before the export URI is constructed.**
The SAS token expiry must account for the full expected export duration plus download time. Generating the token after the URI is constructed, or with too tight an expiry, will cause the export to fail mid-operation with a storage authorization error.

**3. Storage account minimum TLS was at version 1.0.**
The created storage account defaulted to `minimumTlsVersion: TLS1_0`. This is a security gap. In hardened environments, this should be explicitly set to `TLS1_2` at creation time using `--min-tls-version TLS1_2`. This was not required to complete the task but represents a finding that would be flagged in a security review.

**4. Blob tier defaults to Hot.**
The exported BACPAC landed in the Hot access tier. For backup artifacts that are intended for long-term retention and infrequent access, transitioning to Cool or Archive tier after validation significantly reduces storage costs. Implement a lifecycle management policy on the container.

**5. File size consistency is a critical integrity indicator.**
The blob length (`2771` bytes) matched the local download size (`2.8K`). In production, augment this with MD5 checksum verification. The blob response includes a `contentMd5` field (`WYePAZoMyXd7U6yYXOrGEw==`) which can be decoded and compared against the local file checksum to confirm bit-for-bit integrity.

```bash
# Example integrity check using the blob's base64 MD5
md5sum /opt/xfusion-db-backup.bacpac
# Compare decoded base64 of contentMd5 against output
```

**6. Resource group region vs. resource region mismatch.**
The resource group was located in `eastus` while all new resources were provisioned in `westus`. This is valid but introduces latency in cross-region operations and may incur data transfer costs. Align resource group and resource regions in greenfield deployments.

---

## Reference

| Resource | Detail |
|---|---|
| Azure SQL Server | `xfusion-server-3853.database.windows.net` |
| Database | `xfusion-sqldb` (Basic, 5 DTU, 2 GiB) |
| Storage Account | `xfusionst20759` (Standard LRS, StorageV2) |
| Blob Container | `xfusion-container-22348` |
| Backup Artifact | `xfusion-db-backup.bacpac` (2771 bytes) |
| Local Path | `/opt/xfusion-db-backup.bacpac` |
| Subscription | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Resource Group | `kml_rg_main-121b3e04c4694db4` |
| Region | `westus` |
| Export Duration | ~4 minutes (queued: 05:19:14Z, completed: 05:23:22Z) |

---
