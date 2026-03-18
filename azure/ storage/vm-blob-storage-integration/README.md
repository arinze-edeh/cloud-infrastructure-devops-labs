# Azure VM to Blob Storage Integration Lab

> **Enterprise DevOps Infrastructure | Azure Cloud | Storage Automation**

![Azure](https://img.shields.io/badge/Azure-Cloud-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![CLI](https://img.shields.io/badge/Azure_CLI-Automated-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-success?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Environment Details](#environment-details)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Verify Azure Account and VM](#step-1-verify-azure-account-and-vm)
  - [Step 2: Create Private Storage Account](#step-2-create-private-storage-account)
  - [Step 3: Retrieve Storage Account Key](#step-3-retrieve-storage-account-key)
  - [Step 4: Create Blob Container](#step-4-create-blob-container)
  - [Step 5: Connect to the Azure VM via SSH](#step-5-connect-to-the-azure-vm-via-ssh)
  - [Step 6: Create a Test File on the VM](#step-6-create-a-test-file-on-the-vm)
  - [Step 7: Upload File to Blob Storage](#step-7-upload-file-to-blob-storage)
  - [Step 8: Verify Blob Upload](#step-8-verify-blob-upload)
- [Validation and Verification](#validation-and-verification)
- [Security Configuration](#security-configuration)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)
- [Resources](#resources)

---

## Overview

This project documents the end-to-end implementation of a secure Azure infrastructure that enables a Virtual Machine (VM) to interact with Azure Blob Storage for automated file storage and retrieval. The setup follows enterprise-grade security standards using locally redundant storage (LRS), private blob containers, and key-based access control.

This lab was executed as part of the **Nautilus DevOps Team** cloud infrastructure initiative and serves as a foundational pattern for VM-to-storage integrations in production environments.

---

## Problem Statement

**Context:** The Nautilus DevOps team required a reliable, secure, and auditable mechanism for an Azure VM to write data directly to Azure Blob Storage.

**Challenges addressed:**

* No pre-existing storage account or blob container was provisioned for the VM workload
* Public blob access had to be disabled to meet internal security policy
* The VM needed CLI-level access to upload files without relying on managed identity (initial lab scope)
* A repeatable, documented process was needed for onboarding future team members

**Resolution:** A private Azure Storage Account with a dedicated Blob Container was provisioned via Azure CLI. The VM was configured to authenticate using a storage account key, and a test file upload was successfully validated end-to-end.

---

## Architecture

```
+---------------------------+          +-------------------------------+
|     Azure Subscription    |          |     Azure Resource Group      |
|  f0c3bcdd-5ce2-4fa0-...   |          |  KML_RG_MAIN-4E41678F62174ED3 |
+---------------------------+          +-------------------------------+
                                                      |
              +---------------------------------------+---------------------------------------+
              |                                                                               |
+---------------------------+                                         +------------------------+
|      Azure VM             |  SSH (Port 22)                          |   Azure Blob Storage   |
|  Name: devops-vm          | <----+                                  |  devopsstor16287       |
|  OS:   Ubuntu 22.04 LTS   |      |                                  |  SKU: Standard_LRS     |
|  IP:   20.115.8.222       |      |  az storage blob upload          |  Kind: StorageV2       |
|  Region: East US          | +----+----------------------------->    |  Region: East US       |
+---------------------------+                                         |                        |
                                                                      |  Container:            |
                                                                      |  devops-container16287 |
                                                                      |  Access: Private       |
                                                                      +------------------------+
```

---

## Prerequisites

Before beginning this lab, ensure the following are in place:

| Requirement | Details |
|---|---|
| Azure Subscription | Active subscription with Contributor or Owner role |
| Azure CLI | Installed and authenticated (`az login` or Service Principal) |
| SSH Client | Available on your local machine or Azure Cloud Shell |
| Resource Group | `KML_RG_MAIN-4E41678F62174ED3` must exist |
| VM Access | SSH key pair or password-based access to `devops-vm` |

---

## Environment Details

| Parameter | Value |
|---|---|
| Subscription ID | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Tenant ID | `54c1a2d3-d100-453c-9636-3a109eb45552` |
| Resource Group | `KML_RG_MAIN-4E41678F62174ED3` |
| VM Name | `devops-vm` |
| VM Region | `eastus` |
| VM Public IP | `20.115.8.222` |
| VM Private IP | `10.0.0.4` |
| OS | Ubuntu 22.04.5 LTS |
| Storage Account | `devopsstor16287` |
| Container Name | `devops-container16287` |
| Storage SKU | `Standard_LRS` |
| Storage Kind | `StorageV2` |
| Replication | Locally Redundant Storage (LRS) |
| Public Blob Access | Disabled |
| HTTPS Only | Enabled |

---

## Step-by-Step Implementation

### Step 1: Verify Azure Account and VM

Before provisioning any resources, verify the active Azure account context and confirm the target VM exists in the expected resource group.

**Verify active account context:**

```bash
az account show
```

**Expected output snippet:**

```json
{
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "name": "Azure Free Labs",
  "state": "Enabled",
  "tenantId": "54c1a2d3-d100-453c-9636-3a109eb45552"
}
```

**List VMs in the subscription:**

```bash
az vm list --output table
```

**Expected output:**

```
Name       ResourceGroup                 Location    Zones
---------  ----------------------------  ----------  -------
devops-vm  KML_RG_MAIN-4E41678F62174ED3  eastus
```

> **Screenshot**

<img width="1053" height="459" alt="image" src="https://github.com/user-attachments/assets/0feaec15-a1b5-42e5-8f61-c704a1fb273f" />

> *Caption: Azure CLI output confirming active subscription and devops-vm existence in East US*

---

### Step 2: Create Private Storage Account

A new Azure Storage Account is created with public blob access explicitly disabled. This ensures all data in the container is inaccessible without a valid key or SAS token.

```bash
az storage account create \
  --name devopsstor16287 \
  --resource-group KML_RG_MAIN-4E41678F62174ED3 \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --allow-blob-public-access false
```

**Key parameters explained:**

| Parameter | Value | Reason |
|---|---|---|
| `--sku Standard_LRS` | Standard Locally Redundant | Cost-effective for dev/lab workloads |
| `--kind StorageV2` | General Purpose v2 | Supports all storage services and latest features |
| `--allow-blob-public-access false` | Disabled | Enforces private container access |
| `--location eastus` | East US | Co-location with VM to minimize latency |

**Successful provisioning confirms:**

```json
{
  "provisioningState": "Succeeded",
  "allowBlobPublicAccess": false,
  "enableHttpsTrafficOnly": true,
  "name": "devopsstor16287"
}
```

> **Screenshots**

<img width="1032" height="861" alt="image" src="https://github.com/user-attachments/assets/df35eb4f-ef61-4fb7-8ac5-a383cf83b911" />
<img width="1029" height="855" alt="image" src="https://github.com/user-attachments/assets/a6645a04-add2-4b2e-8301-e8c686299996" />
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/f7cb021e-bf60-4de0-b1ca-95d28c212a37" />

> *Caption: Azure CLI confirming successful creation of devopsstor16287 with provisioningState: Succeeded*

---

### Step 3: Retrieve Storage Account Key

The primary access key is retrieved from the newly created storage account and stored in an environment variable for use in subsequent commands. This avoids hardcoding credentials in shell history.

```bash
ACCOUNT_KEY=$(az storage account keys list \
  --account-name devopsstor16287 \
  --resource-group KML_RG_MAIN-4E41678F62174ED3 \
  --query "[0].value" \
  -o tsv)

echo $ACCOUNT_KEY
```

**Output:**

```
eiaftj3MVvdMiB+iKS1hCPUF/3WvrIg40WoIjXPU05jra1RWyetqCi4LhFzPaia8oG4A8KB3vaJ++AStI51hEQ==
```

> **Screenshot**

<img width="1032" height="347" alt="image" src="https://github.com/user-attachments/assets/16fbdbd7-c6ac-4f97-b3e0-58e78efb3d18" />

> *Caption: Storage account key successfully retrieved and stored in the ACCOUNT_KEY environment variable*

> **Security Note:** In production environments, storage account keys should be rotated regularly and managed via Azure Key Vault. Avoid logging or committing keys to version control.

---

### Step 4: Create Blob Container

A private Blob container is created within the storage account. The `--public-access off` flag ensures no anonymous read access is permitted at the container or blob level.

```bash
az storage container create \
  --name devops-container16287 \
  --account-name devopsstor16287 \
  --account-key $ACCOUNT_KEY \
  --public-access off
```

**Successful output:**

```json
{
  "created": true
}
```

> **Screenshot**
> `![Step 4 - Blob Container Creation](./screenshots/step4-blob-container-create.png)`
> *Caption: Blob container devops-container16287 created successfully with public access disabled*

---

### Step 5: Connect to the Azure VM via SSH

The public IP of `devops-vm` is retrieved programmatically and used to establish an SSH session. This ensures the workflow is scriptable and does not depend on manually recorded IP addresses.

**Retrieve the public IP address:**

```bash
PUBLIC_IP=$(az vm list-ip-addresses \
  --name devops-vm \
  --resource-group KML_RG_MAIN-4E41678F62174ED3 \
  --query "[0].virtualMachine.network.publicIpAddresses[0].ipAddress" \
  -o tsv)

echo $PUBLIC_IP
```

**Output:**

```
20.115.8.222
```

**SSH into the VM:**

```bash
ssh azureuser@$PUBLIC_IP
```

**Expected welcome output:**

```
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1044-azure x86_64)

System load:  0.04              Processes:             107
Usage of /:   8.6% of 28.89GB   Users logged in:       0
Memory usage: 37%               IPv4 address for eth0: 10.0.0.4
```

> **Screenshot Placeholder**
> `![Step 5 - SSH Connection to VM](./screenshots/step5-ssh-vm-connect.png)`
> *Caption: Successful SSH connection to devops-vm at 20.115.8.222 running Ubuntu 22.04 LTS*

---

### Step 6: Create a Test File on the VM

Once inside the VM, a test file is created in the `azureuser` home directory. This simulates a real-world scenario where the VM generates or processes a file that must be persisted to cloud storage.

```bash
echo "this is a test file" > /home/azureuser/testfile.txt
cat /home/azureuser/testfile.txt
```

**Output:**

```
this is a test file
```

> **Screenshot Placeholder**
> `![Step 6 - Test File Creation on VM](./screenshots/step6-test-file-create.png)`
> *Caption: testfile.txt created in /home/azureuser with expected content verified via cat*

---

### Step 7: Upload File to Blob Storage

The test file is uploaded from the VM to the Azure Blob container using the Azure CLI `blob upload` command, authenticated using the storage account key.

```bash
az storage blob upload \
  --account-name devopsstor16287 \
  --account-key eiaftj3MVvdMiB+iKS1hCPUF/3WvrIg40WoIjXPU05jra1RWyetqCi4LhFzPaia8oG4A8KB3vaJ++AStI51hEQ== \
  --container-name devops-container16287 \
  --name testfile.txt \
  --file /home/azureuser/testfile.txt
```

**Successful upload output:**

```
Finished[#############################################################]  100.0000%
{
  "client_request_id": "88d49d24-2280-11f1-8942-5d0a2b1ffb81",
  "content_md5": "QiHQAs6108npE35JXOqmRw==",
  "date": "2026-03-18T04:11:26+00:00",
  "etag": "\"0x8DE84A46D116B70\"",
  "lastModified": "2026-03-18T04:11:27+00:00",
  "request_server_encrypted": true,
  "version": "2026-02-06"
}
```

> **Screenshot Placeholder**
> `![Step 7 - Blob Upload from VM](./screenshots/step7-blob-upload.png)`
> *Caption: testfile.txt successfully uploaded to devops-container16287 with 100% progress and server-side encryption confirmed*

---

### Step 8: Verify Blob Upload

After the upload, the contents of the container are listed to confirm the file exists with the correct metadata.

```bash
az storage blob list \
  --account-name devopsstor16287 \
  --account-key eiaftj3MVvdMiB+iKS1hCPUF/3WvrIg40WoIjXPU05jra1RWyetqCi4LhFzPaia8oG4A8KB3vaJ++AStI51hEQ== \
  --container-name devops-container16287 \
  --output table
```

**Confirmed output:**

```
Name          Blob Type    Blob Tier    Length    Content Type    Last Modified              Snapshot
------------  -----------  -----------  --------  --------------  -------------------------  ----------
testfile.txt  BlockBlob    Hot          20        text/plain      2026-03-18T04:11:27+00:00
```

> **Screenshot Placeholder**
> `![Step 8 - Blob List Verification](./screenshots/step8-blob-list-verify.png)`
> *Caption: testfile.txt confirmed present in devops-container16287 as a BlockBlob with correct content type and size*

---

## Validation and Verification

The following checklist confirms all components were provisioned and validated successfully:

| Checkpoint | Status | Notes |
|---|---|---|
| Azure account context verified | PASS | Service Principal authenticated |
| `devops-vm` found in East US | PASS | Existing VM in target resource group |
| Storage account `devopsstor16287` created | PASS | `provisioningState: Succeeded` |
| `allowBlobPublicAccess` set to `false` | PASS | Confirmed in creation output |
| `enableHttpsTrafficOnly` is `true` | PASS | Default and confirmed |
| Storage account key retrieved | PASS | Stored in environment variable |
| Blob container `devops-container16287` created | PASS | `created: true` |
| Container public access set to `off` | PASS | Applied at creation |
| SSH connection to VM established | PASS | Ubuntu 22.04 LTS, IP `20.115.8.222` |
| `testfile.txt` created on VM | PASS | Content verified with `cat` |
| File uploaded to Blob Storage | PASS | `100.0000%` progress, encrypted |
| File visible in container listing | PASS | Correct type, size, and timestamp |

---

## Security Configuration

The following security controls were applied throughout this implementation:

### Storage Account Security

* **Public blob access disabled** (`--allow-blob-public-access false`): Prevents anonymous read access to any blob in the account
* **HTTPS-only traffic** (`enableHttpsTrafficOnly: true`): All data in transit is encrypted via TLS
* **Server-side encryption** (`request_server_encrypted: true`): All blobs are encrypted at rest using Microsoft-managed keys
* **Minimum TLS version**: TLS 1.0 was noted in the lab (see Lessons Learned for recommended hardening)
* **No cross-tenant replication** (`allowCrossTenantReplication: false`): Data is confined to the provisioned tenant

### Network Security

* **Private container**: Container access is restricted to authenticated requests only
* **No public network access override**: Default network rules allow Azure services but no IP-based exceptions were added
* **VM access via SSH**: Key fingerprint verified at first connection

---

## Best Practices

The following best practices apply to this pattern in production environments:

### Identity and Access Management

* **Prefer Managed Identity over Account Keys**: Assign a system-assigned or user-assigned managed identity to the VM and grant it the `Storage Blob Data Contributor` role on the container. This eliminates the need to manage or rotate storage keys manually.
* **Use Azure RBAC for fine-grained access**: Avoid account key usage in production; use role-based access control scoped to the container level.
* **Rotate storage keys periodically**: If keys must be used, rotate them on a schedule using Azure Automation or Key Vault.

### Storage Account Hardening

* **Set minimum TLS version to 1.2**: The lab noted `TLS1_0` as the minimum; this should be updated to `TLS1_2` in all environments:
  ```bash
  az storage account update \
    --name devopsstor16287 \
    --resource-group KML_RG_MAIN-4E41678F62174ED3 \
    --min-tls-version TLS1_2
  ```
* **Enable Soft Delete for blobs**: Protects against accidental or malicious deletion:
  ```bash
  az storage blob service-properties delete-policy update \
    --account-name devopsstor16287 \
    --enable true \
    --days-retained 7
  ```
* **Lock down network access**: Restrict storage account access to the VM's virtual network using service endpoints or private endpoints.
* **Enable Azure Defender for Storage**: Provides anomaly detection and threat intelligence for storage operations.

### Scripting and Automation

* **Store credentials in environment variables or Key Vault**: Never hardcode storage keys in scripts or pipelines.
* **Use `--query` and `-o tsv`** for scripted value extraction to avoid manual copy-paste errors.
* **Validate file existence before upload**: Add a pre-check in automation scripts:
  ```bash
  [ -f /home/azureuser/testfile.txt ] && echo "File exists" || echo "File missing"
  ```
* **Implement upload retry logic**: For production pipelines, wrap blob upload calls with retry logic to handle transient network errors.

### Naming and Tagging

* **Use consistent naming conventions**: Storage account names must be globally unique and lowercase alphanumeric. Use a pattern like `<project><env><random>` (e.g., `devopsstor16287`).
* **Apply resource tags**: Tag all resources with owner, environment, project, and cost-center metadata for governance and cost management:
  ```bash
  az storage account update \
    --name devopsstor16287 \
    --resource-group KML_RG_MAIN-4E41678F62174ED3 \
    --tags environment=lab project=nautilus-devops owner=team-cloud
  ```

---

## Lessons Learned

The following observations were captured during this implementation and should be applied to future Azure storage provisioning work:

### 1. TLS Minimum Version Defaults to TLS 1.0

**Observation:** The newly created storage account had `minimumTlsVersion` set to `TLS1_0` by default.

**Impact:** This exposes the account to connections using outdated TLS versions which are vulnerable to known protocol attacks.

**Action:** Always explicitly set `--min-tls-version TLS1_2` during storage account creation or apply it immediately post-creation via `az storage account update`.

---

### 2. Account Key in Shell History is a Security Risk

**Observation:** The `ACCOUNT_KEY` value was echoed to the terminal and later used inline in `az storage blob upload` and `az storage blob list` commands, which means it may persist in `.bash_history` on the VM.

**Impact:** Any user with access to the VM's shell history can read the storage account key.

**Action:** For production workloads, use Azure Managed Identity. For key-based access in controlled environments, clear bash history after the session or use `HISTCONTROL=ignorespace` to prevent specific commands from being recorded. Consider using `az storage blob upload` with `--auth-mode login` when managed identity is configured.

---

### 3. Co-location of VM and Storage Reduces Latency and Egress Cost

**Observation:** Both `devops-vm` and `devopsstor16287` were provisioned in `eastus`.

**Impact (positive):** VM-to-blob transfers within the same region do not incur egress charges and have lower latency than cross-region transfers.

**Action:** Always co-locate compute and storage resources in the same Azure region unless replication or disaster recovery requires otherwise.

---

### 4. `az vm list-ip-addresses` Enables Fully Scriptable Workflows

**Observation:** Instead of hardcoding the VM's public IP, the IP was retrieved dynamically using `az vm list-ip-addresses` with a `--query` filter.

**Impact (positive):** This pattern is resilient to IP address changes (e.g., VM deallocated and reallocated) and is safe for use in CI/CD pipelines.

**Action:** Always retrieve dynamic Azure resource properties (IPs, keys, connection strings) programmatically rather than hardcoding them in scripts.

---

### 5. Server-Side Encryption is Enabled by Default

**Observation:** The upload response confirmed `"request_server_encrypted": true` without any additional configuration.

**Impact (positive):** All blobs are encrypted at rest using Microsoft-managed keys by default (AES-256).

**Action:** For workloads requiring customer-managed keys (CMK), configure Key Vault integration during or after storage account creation. Do not assume the default encryption model meets all compliance requirements.

---

### 6. Blob Tier Defaults to Hot Access Tier

**Observation:** The blob listing confirmed the uploaded file was placed in the `Hot` access tier automatically.

**Impact:** Hot tier is optimized for frequently accessed data but has a higher storage cost per GB than Cool or Archive tiers.

**Action:** For infrequently accessed blobs (logs, backups, archives), set the access tier to `Cool` or `Archive` to reduce storage costs:
```bash
az storage blob set-tier \
  --account-name devopsstor16287 \
  --container-name devops-container16287 \
  --name testfile.txt \
  --tier Cool
```

---

## Troubleshooting

| Issue | Cause | Resolution |
|---|---|---|
| `AuthorizationFailure` on blob upload | Incorrect or expired account key | Re-retrieve key using `az storage account keys list` |
| `ResourceNotFound` on container creation | Storage account not yet provisioned or wrong name | Verify account name with `az storage account show` |
| SSH connection refused | NSG blocking port 22 or VM deallocated | Check NSG rules and VM power state in Azure Portal |
| `BlobAlreadyExists` on upload | Blob with same name exists in container | Add `--overwrite true` flag to the upload command |
| Storage account name already taken | Names are globally unique across all Azure tenants | Append a unique suffix (timestamp, random number) to the name |
| `PublicAccessNotPermitted` on blob URL | Public access disabled at account level (correct behavior) | Use SAS token, account key, or managed identity for access |

---

## Resources

* [Azure Blob Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
* [Azure CLI Storage Reference](https://learn.microsoft.com/en-us/cli/azure/storage)
* [Azure Managed Identity for VMs](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/tutorial-vm-managed-identities-windows)
* [Azure Storage Security Guide](https://learn.microsoft.com/en-us/azure/storage/common/storage-security-guide)
* [Azure RBAC for Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-auth-aad-rbac-portal)
* [Azure Key Vault + Storage Account Keys](https://learn.microsoft.com/en-us/azure/key-vault/secrets/overview-storage-keys)

---

Environment: Azure Free Labs | East US Region

---






<img width="1035" height="378" alt="image" src="https://github.com/user-attachments/assets/477f91de-af0d-4b8f-ab17-747cdc634517" />
<img width="1032" height="517" alt="image" src="https://github.com/user-attachments/assets/9d11e3e5-15e1-47f8-9e15-694957ba493a" />
<img width="1028" height="872" alt="image" src="https://github.com/user-attachments/assets/1334694b-d2cd-490f-ae05-f66a1c41477f" />
<img width="1034" height="840" alt="image" src="https://github.com/user-attachments/assets/9c99460a-d144-4433-bc3d-1d4ba96e9f91" />
<img width="1032" height="505" alt="image" src="https://github.com/user-attachments/assets/2e11f746-16bf-40d6-af07-958d1b1b1112" />
<img width="1030" height="653" alt="image" src="https://github.com/user-attachments/assets/3fcf6eff-8a11-4106-95e1-505c75c001f0" />
