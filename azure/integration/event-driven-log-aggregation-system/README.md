# Azure Log Pipeline: VM to Event Hubs with Blob Storage Backup

![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=for-the-badge)

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Infrastructure Provisioning](#infrastructure-provisioning)
  - [Step 1: Retrieve the Resource Group](#step-1-retrieve-the-resource-group)
  - [Step 2: Create the Event Hubs Namespace](#step-2-create-the-event-hubs-namespace)
  - [Step 3: Create the Event Hub](#step-3-create-the-event-hub)
  - [Step 4: Create the Storage Account](#step-4-create-the-storage-account)
  - [Step 5: Enable Public Blob Access and Create Container](#step-5-enable-public-blob-access-and-create-container)
  - [Step 6: Provision the Virtual Machine](#step-6-provision-the-virtual-machine)
- [Application Deployment](#application-deployment)
  - [Step 7: Deploy the Log Sender Script to the VM](#step-7-deploy-the-log-sender-script-to-the-vm)
  - [Step 8: Install Python Dependencies on the VM](#step-8-install-python-dependencies-on-the-vm)
  - [Step 9: Inject Connection Strings into the Script](#step-9-inject-connection-strings-into-the-script)
  - [Step 10: Execute the Log Pipeline](#step-10-execute-the-log-pipeline)
- [Verification](#verification)
  - [Step 11: Verify Blob Storage Backup](#step-11-verify-blob-storage-backup)
  - [Step 12: Verify Event Hub Ingestion via Metrics](#step-12-verify-event-hub-ingestion-via-metrics)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Security Considerations](#security-considerations)
- [References](#references)

---

## Overview

This runbook documents the end-to-end provisioning and configuration of a **centralized log collection and backup pipeline** on Microsoft Azure. The solution captures application logs from an Azure Virtual Machine, publishes them to **Azure Event Hubs** for real-time stream processing, and simultaneously archives them to **Azure Blob Storage** for durable backup.

This pattern is widely adopted in enterprise environments for audit trails, compliance archival, and event-driven observability pipelines.

---

## Architecture

```
+------------------+       send_logs.py        +----------------------+
|  Azure VM        |  -----------------------> |  Azure Event Hubs    |
|  (xfusion-vm)    |                           |  (xfusion-namespace) |
|  Ubuntu 22.04    |  -----------------------> |  xfusion-hub         |
+------------------+       (backup)            +----------------------+
                                |
                                v
                    +---------------------------+
                    |  Azure Blob Storage        |
                    |  xfusionst6179             |
                    |  Container:                |
                    |  xfusion-backup-23374      |
                    +---------------------------+
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Full architecture diagram or Azure Portal resource group overview showing all 3 resources]`

---

## Problem Statement

The Nautilus DevOps team required a centralized log collection and backup solution integrating an Azure Virtual Machine with Azure Event Hubs and Azure Blob Storage. Specifically:

* Application logs generated on the VM needed to be streamed in real-time to Event Hubs.
* The same logs needed to be durably backed up to Blob Storage for retention and compliance.
* A Python-based script (`send_logs.py`) already existed on the client host and needed to be deployed and configured on the VM.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Authenticated and configured |
| Active Azure Subscription | With contributor-level access |
| Resource Group | Pre-existing (dynamically queried) |
| Python Script | `send_logs.py` present at `/root/send_logs.py` on client host |
| SSH Key | Generated or pre-existing at `~/.ssh/id_rsa` |

---

## Infrastructure Provisioning

### Step 1: Retrieve the Resource Group

Before provisioning any resource, the active resource group was retrieved dynamically to ensure all subsequent resources land in the correct scope.

```bash
RG=$(az group list --query "[0].name" -o tsv)
echo $RG
```

**Output:**
```
kml_rg_main-8172d8914a6f4db7
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output showing RG variable assignment and echo result]`

---

### Step 2: Create the Event Hubs Namespace

An Event Hubs namespace named `xfusion-namespace` was created in the **East US** region using the **Standard** pricing tier with auto-inflate enabled to handle throughput spikes automatically.

```bash
az eventhubs namespace create \
  --name xfusion-namespace \
  --resource-group $RG \
  --location eastus \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 10
```

**Key configuration values confirmed in output:**

| Field | Value |
|---|---|
| `name` | `xfusion-namespace` |
| `location` | `eastus` |
| `sku.name` | `Standard` |
| `isAutoInflateEnabled` | `true` |
| `maximumThroughputUnits` | `10` |
| `provisioningState` | `Succeeded` |
| `status` | `Active` |

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing full az eventhubs namespace create JSON output with provisioningState: Succeeded]`

---

### Step 3: Create the Event Hub

Within the namespace, an Event Hub named `xfusion-hub` was created. This is the actual message ingestion endpoint that the Python script publishes to.

```bash
az eventhubs eventhub create \
  --name xfusion-hub \
  --namespace-name xfusion-namespace \
  --resource-group $RG
```

**Key configuration values confirmed in output:**

| Field | Value |
|---|---|
| `name` | `xfusion-hub` |
| `location` | `eastus` |
| `partitionCount` | `4` |
| `messageRetentionInDays` | `7` |
| `status` | `Active` |

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing az eventhubs eventhub create JSON output including partitionCount and status: Active]`

---

### Step 4: Create the Storage Account

A general-purpose v2 Storage Account named `xfusionst6179` was created in East US with **Standard_LRS** (Locally Redundant Storage) for cost-effective log archival.

```bash
az storage account create \
  --name xfusionst6179 \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS
```

**Key configuration values confirmed in output:**

| Field | Value |
|---|---|
| `name` | `xfusionst6179` |
| `location` | `eastus` |
| `sku.name` | `Standard_LRS` |
| `kind` | `StorageV2` |
| `provisioningState` | `Succeeded` |
| `allowBlobPublicAccess` | `false` (initial) |
| `enableHttpsTrafficOnly` | `true` |

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing az storage account create JSON response with provisioningState: Succeeded]`

---

### Step 5: Enable Public Blob Access and Create Container

#### 5a. Retrieve the Storage Account Key

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name xfusionst6179 \
  --resource-group $RG \
  --query "[0].value" -o tsv)
```

#### 5b. First Container Creation Attempt (Failed)

The container creation was attempted before public blob access was enabled on the storage account. This resulted in a failed creation.

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Output:**
```json
{
  "created": false
}
```

> **Root Cause:** `allowBlobPublicAccess` was `false` by default on the storage account. A container with `--public-access blob` cannot be created when the account-level public access is disabled.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing container create with created: false result]`

#### 5c. Resolution: Enable Public Blob Access on the Storage Account

```bash
az storage account update \
  --name xfusionst6179 \
  --resource-group $RG \
  --allow-blob-public-access true
```

**Confirmed in output:**
```json
"allowBlobPublicAccess": true
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing az storage account update output with allowBlobPublicAccess: true]`

#### 5d. Retry Container Creation (Succeeded)

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Output:**
```json
{
  "created": true
}
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing container create with created: true result after enabling public access]`

---

### Step 6: Provision the Virtual Machine

An Ubuntu 22.04 LTS Virtual Machine named `xfusion-vm` was provisioned in East US using a `Standard_B1s` compute SKU with a 64 GB OS disk.

```bash
az vm create \
  --name xfusion-vm \
  --resource-group $RG \
  --location eastus \
  --image Ubuntu2204 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --size Standard_B1s \
  --os-disk-size-gb 64 \
  --storage-sku os=Standard_LRS
```

**Output:**
```json
{
  "fqdns": "",
  "id": "/subscriptions/.../virtualMachines/xfusion-vm",
  "location": "eastus",
  "macAddress": "60-45-BD-EE-76-46",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "20.124.200.51",
  "resourceGroup": "kml_rg_main-8172d8914a6f4db7",
  "zones": ""
}
```

The public IP was captured for subsequent SSH operations:

```bash
VM_IP="20.124.200.51"
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing az vm create output with powerState: VM running and public IP]`

---

## Application Deployment

### Step 7: Deploy the Log Sender Script to the VM

The `send_logs.py` script was copied from the client host to the VM's home directory using `scp`.

```bash
scp -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no \
  /root/send_logs.py \
  azureuser@$VM_IP:/home/azureuser/
```

**Output:**
```
Warning: Permanently added '20.124.200.51' (ECDSA) to the list of known hosts.
send_logs.py     100%  960     8.2KB/s   00:00
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing scp transfer completion with file size and transfer speed]`

---

### Step 8: Install Python Dependencies on the VM

#### 8a. Update Package Manager and Install pip

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sudo apt-get update -y && sudo apt-get install -y python3-pip"
```

This installed `python3-pip` along with all required build tools (`build-essential`, `gcc`, `g++`, etc.) and upgraded Python 3.10 components to the latest available versions.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing apt-get install completion with python3-pip successfully installed]`

#### 8b. Install Azure SDK Libraries

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "pip3 install azure-eventhub azure-storage-blob"
```

**Successfully installed packages:**

| Package | Version |
|---|---|
| `azure-eventhub` | `5.15.1` |
| `azure-storage-blob` | `12.28.0` |
| `azure-core` | `1.39.0` |
| `isodate` | `0.7.2` |
| `typing-extensions` | `4.15.0` |

> **Note:** `cryptography` and `requests` were already present via system packages and satisfied as dependencies.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing pip3 install completion with Successfully installed packages listed]`

---

### Step 9: Inject Connection Strings into the Script

#### 9a. Retrieve Connection Strings

```bash
EH_CONN=$(az eventhubs namespace authorization-rule keys list \
  --resource-group $RG \
  --namespace-name xfusion-namespace \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString -o tsv)

SA_CONN=$(az storage account show-connection-string \
  --name xfusionst6179 \
  --resource-group $RG \
  --query connectionString -o tsv)

echo "EH: $EH_CONN"
echo "SA: $SA_CONN"
```

**Output (sanitized for documentation):**
```
EH: Endpoint=sb://xfusion-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<KEY>
SA: DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=xfusionst6179;AccountKey=<KEY>;BlobEndpoint=https://xfusionst6179.blob.core.windows.net/;...
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing EH and SA connection string echo output (redact keys before committing)]`

#### 9b. Inject Connection Strings via sed

The script contained placeholder strings that were replaced in-place using `sed` over SSH.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sed -i 's|<Event Hub Connection String>|$EH_CONN|g' /home/azureuser/send_logs.py"

ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sed -i 's|<Blob Storage Connection String>|$SA_CONN|g' /home/azureuser/send_logs.py"
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing sed injection commands executing without errors]`

---

### Step 10: Execute the Log Pipeline

The script was executed remotely via SSH.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "python3 /home/azureuser/send_logs.py"
```

**Output:**
```
Log sent to Event Hub and backed up to Blob Storage.
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing successful execution output: Log sent to Event Hub and backed up to Blob Storage]`

---

## Verification

### Step 11: Verify Blob Storage Backup

The blob container was queried to confirm the log file was successfully written.

```bash
az storage blob list \
  --container-name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --output table
```

**Output:**

| Name | Blob Type | Length | Content Type | Last Modified |
|---|---|---|---|---|
| `logs.txt` | `AppendBlob` | `18` | `application/octet-stream` | `2026-03-27T05:32:41+00:00` |

> The `AppendBlob` type is significant: it confirms the script used the `AppendBlobClient`, which is the correct blob type for log streaming as it supports appending without overwriting existing content.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing az storage blob list table output with logs.txt present]`

---

### Step 12: Verify Event Hub Ingestion via Metrics

Azure Monitor metrics were queried to confirm that the Event Hub received the message.

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-8172d8914a6f4db7/providers/Microsoft.EventHub/namespaces/xfusion-namespace" \
  --metric "IncomingMessages" \
  --output table
```

**Output (excerpt showing the successful ingestion):**

```
Timestamp             Name               Total
--------------------  -----------------  -------
...
2026-03-27T05:34:00Z  Incoming Messages  1.0
```

> All timestamps prior to `05:34:00Z` showed `0.0`. The single message sent by `send_logs.py` registered as `1.0` at `05:34:00Z`, confirming end-to-end delivery to the Event Hub.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing az monitor metrics list table with IncomingMessages: 1.0 at the bottom timestamp]`

---

## Errors and Resolutions

| Step | Error | Root Cause | Resolution |
|---|---|---|---|
| Step 5b | `"created": false` on container creation | `allowBlobPublicAccess` was `false` by default on the storage account, blocking containers with `--public-access blob` | Updated the storage account with `--allow-blob-public-access true` before retrying the container creation |

> **Key Insight:** Azure Storage Accounts created from March 2023 onward have public blob access disabled by default as a security hardening measure. Any container-level public access setting is subordinate to the account-level setting and will silently fail if the account-level flag is `false`.

---

## Best Practices

### Infrastructure

* **Use dynamic RG resolution** (`az group list --query "[0].name"`) instead of hardcoding resource group names. This makes scripts portable across environments.
* **Enable auto-inflate on Event Hubs** for production workloads. Throughput unit scaling prevents message loss under burst load.
* **Use `Standard_LRS`** for log storage. Logs are non-critical for geo-redundancy since the source VM can regenerate or retransmit; LRS keeps cost minimal.
* **Set `enableHttpsTrafficOnly: true`** (default in newer Azure CLI) to enforce transport encryption for all storage operations.

### Application

* **Use `AppendBlob`** (not `BlockBlob`) for log files. Append blobs support concurrent, sequential writes without overwrite risk, which is the correct semantic for log streaming.
* **Inject secrets at runtime via `sed` or environment variables.** Never commit connection strings into source control. Use Azure Key Vault or environment variable injection in production pipelines.
* **Use `StrictHostKeyChecking=no` only in automation pipelines.** For interactive or production SSH, prefer known-hosts management or Azure Bastion.

### Security

* **Rotate the `RootManageSharedAccessKey`** after initial testing. Create scoped shared access policies with `Send` or `Listen` permissions only for application identities.
* **Prefer Managed Identity over connection strings** for production. Assign the VM a system-assigned managed identity and grant it `Azure Event Hubs Data Sender` and `Storage Blob Data Contributor` roles.
* **Disable public blob access** on storage accounts not requiring public read. The container was made public for this exercise; in production, use private containers with SAS tokens or managed identity access.

### Verification

* **Always verify both the Blob and the Event Hub metric.** A successful `send_logs.py` exit does not guarantee delivery unless both storage artifacts are confirmed.
* **Allow 1 to 2 minutes** for Azure Monitor metrics to reflect recent message counts. Metrics ingestion has a slight delay.

---

## Lessons Learned

1. **Account-level settings gate container-level settings.** The silent `"created": false` on the container is easy to miss. Always verify account-level access flags before configuring container-level settings.

2. **`pip3 install` on a fresh Ubuntu 22.04 VM installs `build-essential` and GCC.** This is expected because `azure-eventhub` compiles C extensions. Budget 2 to 3 minutes for the first `pip3 install` on a minimal VM image.

3. **Azure Monitor metric delay is real.** The `IncomingMessages` metric showed `0.0` for all timestamps up to and including the minute the script ran. It only appeared at `05:34:00Z` after the script completed at approximately `05:32:41Z`. Do not immediately assume failure if the metric counter is zero.

4. **`scp` with `StrictHostKeyChecking=no` adds the host to `known_hosts` automatically.** This is functionally equivalent to accepting the fingerprint interactively. Verify the host fingerprint manually in security-sensitive deployments.

5. **`sed -i` with special characters in connection strings requires careful delimiter selection.** Using `|` as the `sed` delimiter instead of `/` avoids conflicts with forward slashes present in URLs and connection strings.

6. **`AppendBlob` is a write-once-append blob type.** If the container is deleted and recreated, the blob must also be recreated. The `send_logs.py` script likely handles blob creation if it does not exist, which is the correct pattern.

---

## Security Considerations

> **WARNING:** The connection strings and storage account keys shown in this runbook are for documentation purposes. They correspond to a lab environment and should be treated as expired. Never commit real Azure connection strings, access keys, or SAS tokens to source control.

For production deployments, replace connection-string-based authentication with:

```bash
# Assign system-assigned managed identity to the VM
az vm identity assign \
  --name xfusion-vm \
  --resource-group $RG

# Grant Event Hubs Data Sender role
az role assignment create \
  --assignee <vm-principal-id> \
  --role "Azure Event Hubs Data Sender" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.EventHub/namespaces/xfusion-namespace

# Grant Blob Contributor role
az role assignment create \
  --assignee <vm-principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Storage/storageAccounts/xfusionst6179
```

---

## References

* [Azure Event Hubs Documentation](https://learn.microsoft.com/en-us/azure/event-hubs/)
* [Azure Blob Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
* [azure-eventhub Python SDK](https://pypi.org/project/azure-eventhub/)
* [azure-storage-blob Python SDK](https://pypi.org/project/azure-storage-blob/)
* [Azure CLI Reference: az eventhubs](https://learn.microsoft.com/en-us/cli/azure/eventhubs)
* [Azure CLI Reference: az storage](https://learn.microsoft.com/en-us/cli/azure/storage)
* [Azure Monitor Metrics CLI](https://learn.microsoft.com/en-us/cli/azure/monitor/metrics)
* [Managed Identity for Azure VMs](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/qs-configure-cli-windows-vm)

---

## Resource Summary

| Resource | Name | Type | Region | SKU |
|---|---|---|---|---|
| Event Hubs Namespace | `xfusion-namespace` | `Microsoft.EventHub/Namespaces` | East US | Standard |
| Event Hub | `xfusion-hub` | `Microsoft.EventHub/namespaces/eventhubs` | East US | N/A |
| Storage Account | `xfusionst6179` | `Microsoft.Storage/storageAccounts` | East US | Standard_LRS |
| Blob Container | `xfusion-backup-23374` | Blob Container | East US | Public Read |
| Virtual Machine | `xfusion-vm` | `Microsoft.Compute/virtualMachines` | East US | Standard_B1s |

---






<img width="1031" height="403" alt="image" src="https://github.com/user-attachments/assets/1ca491ec-49f9-483e-8cef-b4bd4d10d684" />
<img width="1036" height="841" alt="image" src="https://github.com/user-attachments/assets/2f46273b-8c47-4ff2-9ae6-c727e273110d" />
<img width="1034" height="865" alt="image" src="https://github.com/user-attachments/assets/be149425-5b46-49ad-acc3-d986cc7442bb" />
<img width="1029" height="872" alt="image" src="https://github.com/user-attachments/assets/62146382-dee1-4dd5-9b3a-5d7eae58c2e0" />
<img width="1032" height="866" alt="image" src="https://github.com/user-attachments/assets/1a9578c8-7ff0-4e3b-a4c7-b9acbe501564" />
<img width="1027" height="858" alt="image" src="https://github.com/user-attachments/assets/00096291-17c5-4c12-8b61-77ac3be05aab" />
<img width="1026" height="861" alt="image" src="https://github.com/user-attachments/assets/a8a80c22-b98e-44d5-8c5b-f6a25ec7f4c0" />
<img width="1025" height="870" alt="image" src="https://github.com/user-attachments/assets/21ec68cc-2b72-4e22-b0c8-46ae0e0eb76a" />
<img width="1030" height="863" alt="image" src="https://github.com/user-attachments/assets/d1b7a02e-32be-482c-aa74-078c1c19cd54" />
<img width="1028" height="864" alt="image" src="https://github.com/user-attachments/assets/da04f3f7-1eae-4be8-8dcf-a8ff823a2088" />
<img width="1029" height="853" alt="image" src="https://github.com/user-attachments/assets/2bec3bb9-e537-4c1c-8fd0-fdc8dd3e6c40" />
<img width="1037" height="866" alt="image" src="https://github.com/user-attachments/assets/4ef92411-b5cc-49ed-b767-3b6bdd57ca94" />
<img width="1033" height="687" alt="image" src="https://github.com/user-attachments/assets/6f9ddbb4-f94a-4b86-af43-2f65573f4e96" />
<img width="1034" height="839" alt="image" src="https://github.com/user-attachments/assets/5d4e55d0-1df9-4577-9a3a-ebad1447f819" />
<img width="1033" height="835" alt="image" src="https://github.com/user-attachments/assets/f507d5e8-dd58-4353-90e7-7d3f3129d5dd" />
<img width="1030" height="864" alt="image" src="https://github.com/user-attachments/assets/04732183-96dd-4df0-ba22-021ce105ceff" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/fe2cc2c4-3507-4c58-a6d4-6b8578d1764d" />
<img width="1035" height="647" alt="image" src="https://github.com/user-attachments/assets/8a3e2058-4c5c-42b6-99e8-a29edf151df4" />
<img width="1033" height="542" alt="image" src="https://github.com/user-attachments/assets/9449b667-cd56-405b-bb82-ad6463ddca03" />
<img width="1034" height="606" alt="image" src="https://github.com/user-attachments/assets/e6347d67-9a48-4ea6-b10e-5fac1446177a" />
<img width="1029" height="782" alt="image" src="https://github.com/user-attachments/assets/2b98bf67-b0cf-4a43-9f95-f5ff10dbf262" />
<img width="1034" height="867" alt="image" src="https://github.com/user-attachments/assets/58609aec-57c7-44fe-b6a0-c822a21c3425" />
<img width="1030" height="862" alt="image" src="https://github.com/user-attachments/assets/66bd43e8-21c3-4d17-8341-94bd6337f4ca" />
