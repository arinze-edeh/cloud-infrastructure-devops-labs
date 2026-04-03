# [Azure Blob Storage File Ingestion via CLI](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> **Objective:** Securely upload a local file to an Azure Blob Storage container using the Azure CLI with Azure Active Directory (AAD) authentication, then verify end-to-end data persistence.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Architecture Summary](#architecture-summary)
- [Technologies Used](#technologies-used)
- [Prerequisites](#prerequisites)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Authenticate to Azure](#step-1-authenticate-to-azure)
  - [Step 2: Verify Local File](#step-2-verify-local-file)
  - [Step 3: Discover Storage Account](#step-3-discover-storage-account)
  - [Step 4: Upload File to Blob Container](#step-4-upload-file-to-blob-container)
  - [Step 5: Verify Upload](#step-5-verify-upload)
- [Validation Checklist](#validation-checklist)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices and Lessons Learned](#best-practices-and-lessons-learned)
- [Final Outcome](#final-outcome)

---

## Project Overview

This project demonstrates secure, CLI-driven file ingestion into Azure Blob Storage using AAD-based authentication. The workflow covers identity verification, local asset validation, storage discovery, blob upload, and post-upload confirmation -- reflecting the exact steps required in an enterprise ingestion pipeline.

**Problem statement:** Engineers frequently need to push local files, logs, or artifacts into cloud object storage reliably and securely without exposing storage account keys or SAS tokens in automation scripts.

**Solution:** Use `az storage blob upload` with `--auth-mode login` to leverage the authenticated service principal context, eliminating credential hard-coding while ensuring RBAC-enforced access control.

**Environment details:**

| Resource | Value |
|---|---|
| Storage Account | `xfusionst15842` |
| Blob Container | `xfusion-blob-29364` |
| Region | West US |
| Source File | `/tmp/xfusion.txt` |
| Blob Name | `xfusion.txt` |
| Auth Method | Azure Active Directory (AAD) |

---

## Architecture Summary

```
          ┌─────────────────────────┐
          │  Azure Storage Account  │
          │      xfusionst15842     │
          └───────────┬────────────┘
                      │
          ┌───────────▼────────────┐
          │  Blob Container        │
          │  xfusion-blob-29364    │
          └───────────┬────────────┘
                      │
            ┌─────────▼─────────┐
            │  Block Blob File  │
            │  xfusion.txt      │
            └───────────────────┘
```

The storage account acts as the top-level namespace. The blob container is the logical grouping unit (analogous to a bucket), and the uploaded object is stored as a **Block Blob** -- the optimal type for sequential file uploads under 200 GB.

---

## Technologies Used

- **Azure CLI** -- primary orchestration tool for all resource interactions
- **Azure Blob Storage** -- scalable object storage for unstructured data
- **Azure Active Directory (AAD)** -- identity-based authentication replacing key-based access
- **Linux (Ubuntu/CentOS)** -- host environment for CLI execution

---

## Prerequisites

Before beginning, confirm the following are in place:

- Azure CLI installed and accessible (`az --version`)
- An active Azure account or service principal with the **Storage Blob Data Contributor** role assigned on the target storage account
- The source file exists on the local filesystem (`/tmp/xfusion.txt`)
- The target storage account and blob container are pre-provisioned

---

## Implementation Steps

### Step 1: Authenticate to Azure

Authenticate to Azure using the CLI and confirm the active subscription context before performing any storage operations. Verifying the account identity up front prevents authorization failures mid-operation.

```bash
az login
az account show
```

**Expected output:**

```json
{
  "environmentName": "AzureCloud",
  "homeTenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "isDefault": true,
  "managedByTenants": [],
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "user": {
    "name": "f902723d-a08c-4008-9c35-56cc27e9a966",
    "type": "servicePrincipal"
  }
}
```

**What to verify:**
- `"state": "Enabled"` confirms the subscription is active
- `"isDefault": true` confirms commands will target the correct subscription
- `"type": "servicePrincipal"` confirms non-interactive AAD authentication is in effect

**Operational note:** In CI/CD pipelines, `az login` is replaced by environment variable-based service principal authentication (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`). Always rotate service principal credentials on a defined schedule.

---

**Screenshot -- Azure account authentication confirmed:**

![az account show output confirming authenticated service principal and active subscription](https://github.com/user-attachments/assets/488fecf2-3b9c-4ba1-aa0e-1f067bc76b65)

*`az account show` returns the active subscription context and confirms the service principal identity used for all subsequent CLI operations.*

---

### Step 2: Verify Local File

Before initiating any upload, confirm the source file exists on disk with the expected permissions and size. Uploading a zero-byte or missing file is a silent failure vector that is difficult to debug post-transfer.

```bash
ls -l /tmp/xfusion.txt
```

**Expected output:**

```
-rw-r--r-- 1 root root 33 Feb 25 02:21 /tmp/xfusion.txt
```

**What to verify:**
- File size is non-zero (`33` bytes in this case)
- Permissions allow the executing user to read the file (`r--` for others, `rw-` for root)
- Timestamp reflects expected creation or modification time

**Edge case:** If the file is owned by a different user and read permissions are restricted, the upload will fail with a permission denied error at the CLI layer -- not the Azure layer. Resolve with `chmod +r /tmp/xfusion.txt` before proceeding.

---

**Screenshot -- Local file existence and metadata confirmed:**

![ls -l output confirming xfusion.txt file size, permissions, and timestamp](https://github.com/user-attachments/assets/018e6fe3-79c1-47eb-af59-a720025af1eb)

*The `ls -l` output confirms the file is 33 bytes, readable, and present at the expected path before the upload is attempted.*

---

### Step 3: Discover Storage Account

Query the storage accounts available in the active subscription to confirm the target account name before issuing the upload command. This eliminates typos and confirms scope.

```bash
az storage account list --query "[].name" -o table
```

**Expected output:**

```
Result
--------------
xfusionst15842
```

**What to verify:**
- The target storage account (`xfusionst15842`) appears in the result set
- Only one account is listed, confirming you are operating in the correct subscription scope

**Operational note:** In multi-account environments, extend the query to include the resource group for disambiguation: `--query "[].{Name:name, ResourceGroup:resourceGroup}" -o table`.

---

**Screenshot -- Storage account discovery confirming target account:**

![az storage account list output showing xfusionst15842 as the available storage account](https://github.com/user-attachments/assets/a583967d-4e89-47bb-9280-6f026aaf07a5)

*The storage account list confirms `xfusionst15842` is the active and available target for the blob upload operation.*

---

### Step 4: Upload File to Blob Container

Upload the local file to the pre-provisioned Blob container using AAD-based authentication. The `--auth-mode login` flag enforces identity-based access, ensuring no account keys or SAS tokens are transmitted or stored.

```bash
az storage blob upload \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --name xfusion.txt \
  --file /tmp/xfusion.txt \
  --auth-mode login
```

**Flag breakdown:**

| Flag | Purpose |
|---|---|
| `--account-name` | Identifies the target storage account |
| `--container-name` | Identifies the destination blob container |
| `--name` | Sets the blob object name in the container |
| `--file` | Specifies the local source file path |
| `--auth-mode login` | Authenticates via AAD instead of account key |

**Expected output:**

```json
{
  "client_request_id": "5234cbe2-11f3-11f1-935a-da0a61c5debf",
  "content_md5": "Lu7zilatbGguzSz2Ecn5IQ==",
  "date": "2026-02-25T02:40:19+00:00",
  "encryption_key_sha256": null,
  "encryption_scope": null,
  "etag": "\"0x8DE7417378C459D\"",
  "lastModified": "2026-02-25T02:40:19+00:00",
  "request_id": "ce08f0d1-501e-0002-7500-a69dc1000000",
  "request_server_encrypted": true,
  "version": "2022-11-02",
  "version_id": null
}
```

**What to verify:**
- `"request_server_encrypted": true` confirms Azure encrypted the object at rest on write
- `"content_md5"` provides a checksum for integrity verification post-transfer
- `"lastModified"` timestamp reflects the upload time
- `"etag"` serves as a unique object identifier for conditional request logic

**Risk flag:** If the executing identity lacks the **Storage Blob Data Contributor** role on the container or storage account, this command returns a `403 AuthorizationPermissionMismatch` error. Verify RBAC assignment with `az role assignment list --assignee <principal-id>`.

---

**Screenshot -- Blob upload completed at 100% with server-side encryption confirmed:**

![az storage blob upload output showing 100% progress bar and JSON response confirming upload success and server-side encryption](https://github.com/user-attachments/assets/85fe9add-ba1a-4682-802c-0481995bca75)

*The upload completes at 100% and returns a JSON response. The `request_server_encrypted: true` field confirms Azure Storage encrypted the blob at rest during the write operation.*

---

### Step 5: Verify Upload

List the contents of the blob container to confirm the file was persisted correctly with the expected metadata. This is the final validation gate before declaring the operation complete.

```bash
az storage blob list \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --output table
```

**Expected output:**

| Name | Blob Type | Blob Tier | Length | Content Type | Last Modified | Snapshot |
|---|---|---|---|---|---|---|
| xfusion.txt | BlockBlob | Hot | 33 | text/plain | 2026-02-25T02:40:19+00:00 | |

**What to verify:**
- `Name` matches the intended blob name (`xfusion.txt`)
- `Blob Type` is `BlockBlob` -- appropriate for file uploads
- `Blob Tier` is `Hot` -- correct for frequently accessed or newly created objects
- `Length` matches the source file size (`33` bytes)
- `Content Type` is auto-detected as `text/plain`
- `Last Modified` aligns with the upload timestamp from Step 4

**Operational note:** The CLI warning about missing credentials in the `blob list` command is expected when `--auth-mode login` is omitted. It does not affect output accuracy in this context, but adding `--auth-mode login` to the list command is the recommended practice for consistency.

---

**Screenshot -- Blob container listing confirming successful upload and metadata:**

![az storage blob list output showing xfusion.txt as a BlockBlob in Hot tier with correct size and content type](https://github.com/user-attachments/assets/a3af339f-51ec-420e-ae01-dc6776d8441c)

*The blob list output confirms `xfusion.txt` is present in the container as a BlockBlob, 33 bytes in size, stored in the Hot tier with correct content type and modification timestamp.*

---

## Validation Checklist

| Check | Status |
|---|---|
| Azure login successful and subscription confirmed | ✅ |
| Local source file verified (non-zero, readable) | ✅ |
| Target storage account discovered and confirmed | ✅ |
| Blob uploaded with AAD authentication | ✅ |
| Server-side encryption confirmed on upload response | ✅ |
| Blob visible in container with correct metadata | ✅ |

---

## Key Decisions

**AAD authentication over account key:** Using `--auth-mode login` enforces identity-based access control and eliminates the risk of key leakage in scripts or shell history. This is the recommended approach in any environment subject to compliance requirements (PCI-DSS, SOC 2, ISO 27001).

**Block Blob type:** Azure automatically selects Block Blob for standard file uploads, which is optimal for files up to 190.7 TB and supports parallel chunk uploads for large files. Append Blobs are used for log streaming; Page Blobs for VM disk images.

**Hot access tier:** Newly uploaded blobs default to the Hot tier, which is appropriate for files requiring immediate access. For archival use cases, transition to Cool or Archive tier using `az storage blob set-tier` to reduce storage costs.

**`--name` flag for blob naming:** Explicitly setting `--name xfusion.txt` ensures the blob name in the container is predictable and does not inherit path components from the local filesystem.

---

## Errors and Resolutions

**Missing RBAC assignment (403 AuthorizationPermissionMismatch)**

Occurs when `--auth-mode login` is used but the authenticated principal lacks the **Storage Blob Data Contributor** role on the storage account or container.

Resolution:
```bash
az role assignment create \
  --assignee <principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/xfusionst15842
```

**Missing credentials warning on `blob list`**

The CLI displays a warning when `--auth-mode login` is omitted from listing commands, stating it will fall back to querying the account key. This does not block output but indicates an inconsistency in auth posture.

Resolution: Append `--auth-mode login` to all storage commands for consistency:
```bash
az storage blob list \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --output table \
  --auth-mode login
```

**File not found at upload time**

If the source file path is incorrect or the file has been deleted between verification and upload, the CLI returns a `FileNotFoundError`.

Resolution: Always run `ls -l <path>` immediately before the upload command in automated scripts. In pipelines, use `test -f /tmp/xfusion.txt || exit 1` as a pre-flight gate.

---

## Best Practices and Lessons Learned

- **Always validate the source file before upload.** Silent zero-byte uploads are harder to detect than an explicit pre-flight check failure.
- **Use `--auth-mode login` on every storage command**, not just uploads. Inconsistent auth modes in a script create confusion during audits and debugging.
- **Capture `content_md5` from the upload response** and store it alongside the object reference. This enables integrity verification if the blob is later retrieved or replicated.
- **Avoid storing account keys in shell history or scripts.** Even in non-production environments, key-based auth creates habits that are difficult to unlearn. Defaulting to AAD from the start builds the right muscle memory.
- **Tag blobs at upload time** using `--metadata` or post-upload with `az storage blob metadata update` to support lifecycle management, cost allocation, and automated tiering policies.
- **Monitor blob storage with Azure Monitor** and configure alerts on ingress/egress anomalies and failed authentication attempts as a baseline security posture.

---

## Final Outcome

The file `/tmp/xfusion.txt` was successfully uploaded to Azure Blob Storage container `xfusion-blob-29364` under storage account `xfusionst15842` using AAD-based CLI authentication. The upload was verified through both the upload response metadata (MD5 checksum, etag, server-side encryption confirmation) and a subsequent blob listing confirming persistence with correct size, type, and tier.

This workflow is directly portable to automated ingestion pipelines, CI/CD artifact archival jobs, and scheduled log push operations where key-free, identity-governed storage access is required.




























# Azure Blob Storage File Upload – Nautilus DevOps Lab

## Project Overview

- This project documents the process of uploading a local file to an Azure Blob Storage container using the Azure CLI.

- The file `/tmp/xfusion.txt` was uploaded to the Blob container `xfusion-blob-29364` in the `westus region` under the storage account `xfusionst15842`.

- Objective: Validate end-to-end file upload and verification in Azure Blob Storage.

## Architecture Summary
          ┌─────────────────────────┐
          │  Azure Storage Account  │
          │      xfusionst15842     │
          └───────────┬────────────┘
                      │
          ┌───────────▼────────────┐
          │  Blob Container        │
          │  xfusion-blob-29364    │
          └───────────┬────────────┘
                      │
            ┌─────────▼─────────┐
            │  Block Blob File  │
            │  xfusion.txt      │
            └───────────────────┘

## 🔧 Technologies Used

- Azure CLI

- Azure Blob Storage

- Linux (Ubuntu/CentOS)

- Azure Active Directory (AD) Authentication

- Public Blob Containers

## 🚀 Implementation Steps

### Step 1️: Login to Azure

- `az login`

Check your account details:

- `az account show`

Expected Output:
````
{
  "environmentName": "AzureCloud",
  "homeTenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512",
  "isDefault": true,
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552",
  "user": {
    "name": "f902723d-a08c-4008-9c35-56cc27e9a966",
    "type": "servicePrincipal"
  }
}
````

📸 Screenshots:
<img width="1031" height="569" alt="image" src="https://github.com/user-attachments/assets/488fecf2-3b9c-4ba1-aa0e-1f067bc76b65" />

### 2️⃣ Verify Local File
- `ls -l /tmp/xfusion.txt`

Expected Output:

- `-rw-r--r-- 1 root root 33 Feb 25 02:21 /tmp/xfusion.txt`

📸 Screenshots:
<img width="1029" height="587" alt="image" src="https://github.com/user-attachments/assets/018e6fe3-79c1-47eb-af59-a720025af1eb" />

### 3️⃣ List Storage Accounts
- `az storage account list --query "[].name" -o table`

Expected Output:
````
Result
--------------
xfusionst15842
````

📸 Screenshots:
<img width="1027" height="640" alt="image" src="https://github.com/user-attachments/assets/a583967d-4e89-47bb-9280-6f026aaf07a5" />

### 4️⃣ Upload File to Blob Container
````
az storage blob upload \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --name xfusion.txt \
  --file /tmp/xfusion.txt \
  --auth-mode login
````

Expected Output:
````
{
  "client_request_id": "5234cbe2-11f3-11f1-935a-da0a61c5debf",
  "content_md5": "Lu7zilatbGguzSz2Ecn5IQ==",
  "date": "2026-02-25T02:40:19+00:00",
  "etag": "\"0x8DE7417378C459D\"",
  "lastModified": "2026-02-25T02:40:19+00:00",
  "request_server_encrypted": true,
  "version": "2022-11-02"
}
````
📸 Screenshots:
<img width="1032" height="862" alt="image" src="https://github.com/user-attachments/assets/85fe9add-ba1a-4682-802c-0481995bca75" />

### 5️⃣ Verify Upload
````
az storage blob list \
  --account-name xfusionst15842 \
  --container-name xfusion-blob-29364 \
  --output table
````
Expected Output:

| Property       | Value                         |
|----------------|-------------------------------|
| Name           | xfusion.txt                   |
| Blob Type      | BlockBlob                     |
| Blob Tier      | Hot                           |
| Length         | 33                            |
| Content Type   | text/plain                    |
| Last Modified  | 2026-02-25T02:40:19+00:00    |

📸 Screenshots:
<img width="1036" height="855" alt="image" src="https://github.com/user-attachments/assets/a3af339f-51ec-420e-ae01-dc6776d8441c" />

### Validation Checklist

| Check                      | Status |
| -------------------------- | ------ |
| Azure login successful     | ✅      |
| Local file exists          | ✅      |
| Storage account exists     | ✅      |
| Blob uploaded successfully | ✅      |
| Blob visible in container  | ✅      |


### Final Outcome

- Successfully uploaded /tmp/xfusion.txt to Azure Blob Storage.

- Verified end-to-end file upload using Azure CLI.

- Data is stored in a public Blob container for further access.
