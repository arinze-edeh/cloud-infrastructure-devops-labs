# Azure Centralized Log Collection and Backup Pipeline

> **Enterprise-grade log ingestion architecture using Azure Event Hubs, Azure Blob Storage, and a Linux Virtual Machine. This pipeline enables real-time event streaming with durable cold-storage backup, following cloud-native DevOps principles.**

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Infrastructure Provisioning](#infrastructure-provisioning)
  - [Step 1 -- Create Azure Event Hubs Namespace and Event Hub](#step-1----create-azure-event-hubs-namespace-and-event-hub)
  - [Step 2 -- Provision Azure Blob Storage Account and Container](#step-2----provision-azure-blob-storage-account-and-container)
  - [Step 3 -- Deploy the Azure Virtual Machine](#step-3----deploy-the-azure-virtual-machine)
  - [Step 4 -- Install Dependencies on the VM](#step-4----install-dependencies-on-the-vm)
  - [Step 5 -- Retrieve Connection Strings](#step-5----retrieve-connection-strings)
  - [Step 6 -- Inject Credentials and Execute the Log Script](#step-6----inject-credentials-and-execute-the-log-script)
  - [Step 7 -- Verify Blob Storage Backup](#step-7----verify-blob-storage-backup)
  - [Step 8 -- Validate Event Hub Metrics](#step-8----validate-event-hub-metrics)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Resource Reference](#resource-reference)

---

## Problem Statement

The Nautilus DevOps team required a centralized, scalable log pipeline to:

- Stream application logs in real-time to **Azure Event Hubs** for downstream processing and analytics.
- Persist log backups to **Azure Blob Storage** for compliance, audit trails, and cold storage access.
- Use a **Linux VM** as the client compute layer, simulating a real-world application host.

The challenge was to provision and wire together all three Azure services, deploy the log-sending script, and validate end-to-end log delivery from a VM to both the Event Hub and the Blob container.

---

## Architecture Overview

```
+------------------+        send_logs.py        +----------------------+
|   Azure Linux VM  | --------------------------> |  Azure Event Hubs    |
|  (xfusion-vm)    |                             |  (xfusion-namespace) |
|  Ubuntu 22.04    | --------+                   |  Hub: xfusion-hub    |
+------------------+         |                   +----------------------+
                              |
                              v
                   +-------------------------+
                   |  Azure Blob Storage     |
                   |  (xfusionst6179)        |
                   |  Container:             |
                   |  xfusion-backup-23374   |
                   |  Blob: logs.txt         |
                   +-------------------------+
```

**Region:** East US
**SKU:** Standard (Event Hubs), Standard_LRS (Storage), Standard_B1s (VM)

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Authenticated session with active subscription |
| Existing Resource Group | Set as `$RG` environment variable |
| Pre-existing Event Hubs Namespace | `xfusion-namespace` (provisioned prior to this runbook) |
| Python script on client host | `/root/send_logs.py` with placeholder connection strings |
| SSH access | Key-based authentication to the VM |

---

## Infrastructure Provisioning

### Step 1 -- Create Azure Event Hubs Namespace and Event Hub

> **Note:** The Event Hubs namespace `xfusion-namespace` was pre-provisioned in the environment. An Event Hub named `xfusion-hub` was created within it using the Azure Portal or CLI prior to this workflow. Connection string retrieval is covered in Step 5.

---

### Step 2 -- Provision Azure Blob Storage Account and Container

#### 2a. Create the Storage Account

```bash
az storage account create \
  --name xfusionst6179 \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS
```

**Expected output (key fields):**

```json
{
  "name": "xfusionst6179",
  "location": "eastus",
  "sku": { "name": "Standard_LRS" },
  "kind": "StorageV2",
  "provisioningState": "Succeeded",
  "allowBlobPublicAccess": false
}
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Azure Portal -- Storage Account xfusionst6179 overview showing provisioningState: Succeeded ]`

---

#### 2b. Retrieve the Storage Account Key

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name xfusionst6179 \
  --resource-group $RG \
  --query "[0].value" -o tsv)
```

---

#### 2c. First Container Creation Attempt (Failed -- Public Access Blocked)

The initial attempt to create the blob container with `--public-access blob` failed because the storage account had `allowBlobPublicAccess` set to `false` by default.

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Output (failure indicator):**

```json
{ "created": false }
```

> This is the first error encountered in the pipeline. See [Errors Encountered and Resolutions](#errors-encountered-and-resolutions) for full details.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing { "created": false } after initial container creation attempt ]`

---

#### 2d. Enable Public Blob Access on Storage Account

```bash
az storage account update \
  --name xfusionst6179 \
  --resource-group $RG \
  --allow-blob-public-access true
```

**Expected output (key change):**

```json
{
  "allowBlobPublicAccess": true,
  "provisioningState": "Succeeded"
}
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal output confirming allowBlobPublicAccess: true after account update ]`

---

#### 2e. Re-create Container (Success)

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Output (success):**

```json
{ "created": true }
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal output showing { "created": true } for container xfusion-backup-23374 ]`

---

### Step 3 -- Deploy the Azure Virtual Machine

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

**Expected output:**

```json
{
  "fqdns": "",
  "location": "eastus",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "20.124.200.51"
}
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Azure Portal -- Virtual Machine xfusion-vm showing "Running" power state in East US ]`

Capture the public IP for subsequent SSH operations:

```bash
VM_IP="20.124.200.51"
```

---

### Step 4 -- Install Dependencies on the VM

#### 4a. Copy the Log Script from the Client Host to the VM

```bash
scp -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no \
  /root/send_logs.py \
  azureuser@$VM_IP:/home/azureuser/
```

**Expected output:**

```
send_logs.py    100%  960     8.2KB/s   00:00
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing successful SCP transfer of send_logs.py to xfusion-vm ]`

---

#### 4b. Update APT and Install Python3 pip

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sudo apt-get update -y && sudo apt-get install -y python3-pip"
```

This step installs `python3-pip` along with all required build toolchains (`gcc`, `g++`, `build-essential`, etc.) as transitive dependencies.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing apt-get install output with python3-pip installation confirmed ]`

---

#### 4c. Install Azure SDK Packages

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "pip3 install azure-eventhub azure-storage-blob"
```

**Packages installed:**

| Package | Version |
|---|---|
| azure-eventhub | 5.15.1 |
| azure-storage-blob | 12.28.0 |
| azure-core | 1.39.0 |
| isodate | 0.7.2 |
| typing-extensions | 4.15.0 |

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing pip3 install success for azure-eventhub and azure-storage-blob ]`

---

### Step 5 -- Retrieve Connection Strings

Both the Event Hub and Blob Storage connection strings are extracted and stored as shell variables for secure inline injection.

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

**Expected output format:**

```
EH: Endpoint=sb://xfusion-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<REDACTED>
SA: DefaultEndpointsProtocol=https;AccountName=xfusionst6179;AccountKey=<REDACTED>;BlobEndpoint=https://xfusionst6179.blob.core.windows.net/;...
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing both EH and SA connection strings printed (keys redacted) ]`

> **Security Note:** In production environments, connection strings must never be echoed to stdout or committed to version control. Use Azure Key Vault references or environment-scoped secret managers instead.

---

### Step 6 -- Inject Credentials and Execute the Log Script

#### 6a. Replace Placeholder Strings in the Script (on the VM)

The `send_logs.py` script ships with two placeholder strings that are substituted using `sed` over SSH, eliminating the need to store credentials locally or re-upload the script.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sed -i 's|<Event Hub Connection String>|$EH_CONN|g' /home/azureuser/send_logs.py"

ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sed -i 's|<Blob Storage Connection String>|$SA_CONN|g' /home/azureuser/send_logs.py"
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal showing both sed substitution SSH commands completing without error ]`

---

#### 6b. Execute the Log Sender Script

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "python3 /home/azureuser/send_logs.py"
```

**Expected output:**

```
Log sent to Event Hub and backed up to Blob Storage.
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal confirming "Log sent to Event Hub and backed up to Blob Storage." ]`

---

### Step 7 -- Verify Blob Storage Backup

```bash
az storage blob list \
  --container-name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --output table
```

**Expected output:**

```
Name      Blob Type    Blob Tier    Length    Content Type              Last Modified
--------  -----------  -----------  --------  ------------------------  -------------------------
logs.txt  AppendBlob                18        application/octet-stream  2026-03-27T05:32:41+00:00
```

This confirms a blob named `logs.txt` of type `AppendBlob` was successfully written to the container.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal table listing logs.txt as AppendBlob in xfusion-backup-23374 container ]`

> **Screenshot Placeholder**
> `[ SCREENSHOT: Azure Portal -- Blob container xfusion-backup-23374 showing logs.txt file object ]`

---

### Step 8 -- Validate Event Hub Metrics

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-8172d8914a6f4db7/providers/Microsoft.EventHub/namespaces/xfusion-namespace" \
  --metric "IncomingMessages" \
  --output table
```

The metric output shows `IncomingMessages` across minute-level time buckets. All entries show `0.0` until the script execution window, where a value of `1.0` is recorded at `2026-03-27T05:34:00Z`, confirming the message was received by the Event Hub.

**Key metric entry:**

```
2026-03-27T05:34:00Z  Incoming Messages  1.0
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal table showing IncomingMessages metric with 1.0 at 05:34:00Z ]`

> **Screenshot Placeholder**
> `[ SCREENSHOT: Azure Portal -- Event Hub namespace xfusion-namespace Metrics blade showing IncomingMessages spike ]`

---

## Errors Encountered and Resolutions

### Error 1 -- Blob Container Creation Failed: `"created": false`

**Step:** 2c

**Root Cause:**
The storage account was created with `allowBlobPublicAccess` defaulting to `false` (a secure Azure default as of recent API versions). Attempting to create a container with `--public-access blob` against an account that prohibits public blob access results in a silent failure where the API returns `{ "created": false }` without an explicit error message.

**Symptom:**
```json
{ "created": false }
```

**Resolution:**
Update the storage account to explicitly allow public blob access before re-attempting the container creation:

```bash
az storage account update \
  --name xfusionst6179 \
  --resource-group $RG \
  --allow-blob-public-access true
```

Then re-run the container creation command. Output changes to `{ "created": true }`.

**Production Recommendation:**
For production workloads, avoid enabling public blob access. Use Shared Access Signatures (SAS tokens), Azure AD-based authentication, or private endpoints to control access without opening the container publicly.

---

## Best Practices

### Credential Management

- Never hardcode connection strings in scripts committed to version control.
- Use **Azure Key Vault** to store and retrieve secrets at runtime.
- For long-lived compute, prefer **Managed Identity** over connection string authentication to eliminate credential rotation overhead.
- Redact secrets in CI/CD pipeline logs using secret masking features.

### Networking and Security

- Restrict public network access to the storage account using **Private Endpoints** or **VNet Service Endpoints**.
- Apply **Network Security Group (NSG)** rules to limit inbound SSH (port 22) to known CIDR ranges only.
- Enable **Azure Defender for Storage** to detect anomalous blob access patterns.
- Set `minimumTlsVersion` to `TLS1_2` on the storage account (the provisioned account defaulted to `TLS1_0` -- a known gap).

### Event Hubs Configuration

- Enable **Auto-inflate** on the Event Hubs namespace to automatically scale throughput units under load.
- Set appropriate **message retention** (1 to 7 days on Standard tier) aligned to downstream consumer SLAs.
- Use **consumer groups** to isolate readers for different downstream services.

### Virtual Machine Hygiene

- Use **cloud-init** or **Azure Custom Script Extension** for idempotent VM bootstrapping instead of interactive SSH commands.
- Store SSH keys in a secrets manager and rotate them on a defined schedule.
- Prefer **Azure Bastion** over direct public IP SSH for production VM access.

### Observability

- Set **Azure Monitor Alerts** on `IncomingMessages` dropping to zero (indicating pipeline stall).
- Export Event Hub metrics to a **Log Analytics Workspace** for dashboarding and alerting.
- Use **Azure Storage Diagnostics** logging to track blob read/write operations for audit purposes.

### Infrastructure as Code

- Replace all manual `az` CLI commands with **Bicep** or **Terraform** templates for repeatability and drift detection.
- Store IaC templates in a dedicated repository branch with pull request review gates.
- Use **Azure Resource Locks** on production resource groups to prevent accidental deletion.

---

## Lessons Learned

**1. Azure secure defaults require explicit overrides.**
New storage accounts disable public blob access by default. While this is the correct security posture, it surfaced as a silent `"created": false` failure. Teams must understand that the Azure CLI will not always throw hard errors on permission conflicts -- defensive validation of output fields is required.

**2. `sed` inline replacement is effective but fragile for secrets.**
Using `sed -i` over SSH to inject connection strings into scripts works in lab environments but introduces risks in production: connection strings may contain characters that break `sed` delimiter patterns (e.g., `/`, `&`). The pipe `|` delimiter was correctly used here as a workaround. In production, use templating engines or secret managers with proper escaping.

**3. Metric propagation in Azure Monitor has latency.**
The `IncomingMessages` metric showed `0.0` for all minute buckets prior to execution. The actual message ingestion at `05:32:41` was only reflected in the metric output at `05:34:00`, indicating approximately 60 to 90 seconds of metric propagation delay. Operators must account for this lag when building alerting thresholds.

**4. pip3 installs to user scope when run over SSH.**
The pip3 install completed with the warning `Defaulting to user installation because normal site-packages is not writeable`. This occurred because the SSH session did not run with the necessary privileges to write to system site-packages. The user-scoped install succeeded, but for system-wide deployments, use `sudo pip3 install` or a virtual environment with proper ownership.

**5. AppendBlob type indicates correct SDK usage.**
The blob `logs.txt` was written as an `AppendBlob`, which is the appropriate type for log files that accumulate over time. This confirms the `azure-storage-blob` SDK was used with the correct blob type API, allowing subsequent log writes to append rather than overwrite.

**6. TLS version default is outdated.**
The storage account provisioned with `minimumTlsVersion: TLS1_0`. This is a security gap that should always be patched immediately post-provisioning by setting `--min-tls-version TLS1_2` in the `az storage account create` command or in a follow-up `az storage account update` call.

---

## Resource Reference

| Resource | Name | Region |
|---|---|---|
| Resource Group | `kml_rg_main-8172d8914a6f4db7` | East US |
| Event Hubs Namespace | `xfusion-namespace` | East US |
| Event Hub | `xfusion-hub` | East US |
| Storage Account | `xfusionst6179` | East US |
| Blob Container | `xfusion-backup-23374` | East US |
| Virtual Machine | `xfusion-vm` | East US |
| VM Image | Ubuntu 22.04 LTS | -- |
| VM Size | Standard_B1s | -- |
| VM Admin User | `azureuser` | -- |
| VM Public IP | `20.124.200.51` | -- |

---

Environment: Azure Cloud (East US)
Authentication: Azure CLI with Subscription `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`

---

*This document follows internal DevOps documentation standards. All credentials shown in terminal outputs are rotated post-execution. Do not commit live connection strings to any version control system.*


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
