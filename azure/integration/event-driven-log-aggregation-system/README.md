# Azure Centralized Log Collection and Backup Pipeline

> **Enterprise-grade log ingestion pipeline using Azure CLI, Azure Event Hubs, Azure Blob Storage, and a Linux Virtual Machine. This runbook documents the exact CLI-driven process to provision infrastructure, deploy a log-forwarding script, and validate end-to-end message delivery from a VM to both an Event Hub and a Blob container.**

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Execution Runbook](#execution-runbook)
  - [Step 1 -- Create the Azure Storage Account](#step-1----create-the-azure-storage-account)
  - [Step 2 -- Retrieve the Storage Account Key and Attempt Container Creation (Failed)](#step-2----retrieve-the-storage-account-key-and-attempt-container-creation-failed)
  - [Step 3 -- Enable Public Blob Access on the Storage Account](#step-3----enable-public-blob-access-on-the-storage-account)
  - [Step 4 -- Re-create the Blob Container (Success)](#step-4----re-create-the-blob-container-success)
  - [Step 5 -- Deploy the Azure Virtual Machine](#step-5----deploy-the-azure-virtual-machine)
  - [Step 6 -- Set the VM Public IP Variable](#step-6----set-the-vm-public-ip-variable)
  - [Step 7 -- Copy the Log Script from Client Host to the VM](#step-7----copy-the-log-script-from-client-host-to-the-vm)
  - [Step 8 -- Install Python3 pip on the VM](#step-8----install-python3-pip-on-the-vm)
  - [Step 9 -- Install Azure SDK Packages on the VM](#step-9----install-azure-sdk-packages-on-the-vm)
  - [Step 10 -- Retrieve Event Hub and Storage Connection Strings](#step-10----retrieve-event-hub-and-storage-connection-strings)
  - [Step 11 -- Inject Event Hub Connection String into the Script](#step-11----inject-event-hub-connection-string-into-the-script)
  - [Step 12 -- Inject Blob Storage Connection String into the Script](#step-12----inject-blob-storage-connection-string-into-the-script)
  - [Step 13 -- Execute the Log Sender Script on the VM](#step-13----execute-the-log-sender-script-on-the-vm)
  - [Step 14 -- Verify the Blob Backup in the Container](#step-14----verify-the-blob-backup-in-the-container)
  - [Step 15 -- Validate Event Hub IncomingMessages Metrics](#step-15----validate-event-hub-incomingmessages-metrics)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Resource Reference](#resource-reference)

---

## Problem Statement

The Nautilus DevOps team required a centralized, scalable log pipeline to:

- Stream application logs in real-time to **Azure Event Hubs** for downstream processing and analytics.
- Persist log backups durably to **Azure Blob Storage** for compliance, audit trails, and cold storage access.
- Use an **Azure Linux VM** as the compute layer simulating a real-world application host.

The entire pipeline was provisioned and validated exclusively via the **Azure CLI** from the client host shell. No Azure Portal interaction was used at any point in this workflow.

---

## Architecture Overview

```
+----------------------+         SCP + SSH          +----------------------+
|   Client Host        | --------------------------> |   Azure Linux VM     |
|   /root/send_logs.py |                             |   xfusion-vm         |
|   ~/.ssh/id_rsa      |                             |   Ubuntu 22.04 LTS   |
+----------------------+                             |   azureuser@20.124.. |
                                                     +----------+-----------+
                                                                |
                                          python3 send_logs.py |
                                                                |
                          +-----------------+-----------------+-+
                          |                                   |
                          v                                   v
             +------------------------+       +--------------------------------+
             |   Azure Event Hubs     |       |   Azure Blob Storage           |
             |   xfusion-namespace    |       |   xfusionst6179                |
             |   (Standard tier)      |       |   Container: xfusion-backup-.. |
             |   IncomingMessages: 1  |       |   Blob: logs.txt (AppendBlob)  |
             +------------------------+       +--------------------------------+
```

**Region:** East US
**Auth method:** Azure CLI (`az`) with pre-authenticated session
**Compute:** Standard_B1s, Ubuntu 22.04 LTS, 64 GB OS disk, Standard_LRS

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Installed and authenticated (`az login` completed prior to this workflow) |
| `$RG` environment variable | Set to the target resource group name |
| Pre-existing Event Hubs Namespace | `xfusion-namespace` already exists in the resource group |
| Pre-existing Event Hub | `xfusion-hub` already created inside `xfusion-namespace` |
| `send_logs.py` on client host | Located at `/root/send_logs.py`; contains two literal placeholders: `<Event Hub Connection String>` and `<Blob Storage Connection String>` |
| SSH key pair | Not pre-existing; auto-generated by `--generate-ssh-keys` during VM creation in Step 5 |

---

## Execution Runbook

### Step 1 -- Create the Azure Storage Account

The storage account `xfusionst6179` was created in the East US region using the Standard_LRS SKU via the Azure CLI.

```bash
az storage account create \
  --name xfusionst6179 \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS
```

**Actual output (abbreviated):**

```json
{
  "accessTier": "Hot",
  "allowBlobPublicAccess": false,
  "creationTime": "2026-03-27T05:17:31.842324+00:00",
  "enableHttpsTrafficOnly": true,
  "kind": "StorageV2",
  "location": "eastus",
  "minimumTlsVersion": "TLS1_0",
  "name": "xfusionst6179",
  "provisioningState": "Succeeded",
  "sku": {
    "name": "Standard_LRS",
    "tier": "Standard"
  }
}
```

> **Key observations from this output:**
> - `allowBlobPublicAccess` is `false` by default. This will directly cause the container creation failure in Step 2.
> - `minimumTlsVersion` defaulted to `TLS1_0`. This is a security gap that must be patched. See [Best Practices](#best-practices).
> - Blob and file service encryption was enabled automatically by Azure (`"enabled": true` under both services).

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- full JSON output of az storage account create confirming provisioningState: Succeeded and allowBlobPublicAccess: false ]`

---

### Step 2 -- Retrieve the Storage Account Key and Attempt Container Creation (Failed)

Both sub-commands were executed in the same command block. First the primary storage key was extracted into `$STORAGE_KEY`, then the container creation was immediately attempted in the same session.

#### 2a. Retrieve the Storage Account Key

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name xfusionst6179 \
  --resource-group $RG \
  --query "[0].value" -o tsv)
```

No output is printed. The key is held in memory as a shell variable.

#### 2b. Attempt Container Creation with Public Access (Failed)

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Actual output:**

```json
{
  "created": false
}
```

**What happened:** The storage account has `allowBlobPublicAccess: false` at the account level (set by default in Step 1). Azure blocks the public-access container creation and returns `"created": false` with no error message and no non-zero exit code. The container was not created.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- STORAGE_KEY assignment followed immediately by az storage container create returning { "created": false } ]`

See [Errors Encountered and Resolutions](#errors-encountered-and-resolutions) for the full root cause analysis and prevention strategy.

---

### Step 3 -- Enable Public Blob Access on the Storage Account

To unblock the container creation, the storage account was updated via `az storage account update` to enable public blob access at the account level.

```bash
az storage account update \
  --name xfusionst6179 \
  --resource-group $RG \
  --allow-blob-public-access true
```

**Actual output (abbreviated):**

```json
{
  "allowBlobPublicAccess": true,
  "name": "xfusionst6179",
  "provisioningState": "Succeeded"
}
```

`allowBlobPublicAccess` is now `true`. Container creation with `--public-access blob` will succeed on the next attempt.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- az storage account update output showing allowBlobPublicAccess: true ]`

---

### Step 4 -- Re-create the Blob Container (Success)

The identical container creation command from Step 2b was re-executed after the account update in Step 3.

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Actual output:**

```json
{
  "created": true
}
```

The container `xfusion-backup-23374` now exists with public blob read access enabled and is ready to receive backups.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- az storage container create returning { "created": true } ]`

---

### Step 5 -- Deploy the Azure Virtual Machine

The VM `xfusion-vm` was created in the same resource group and region using Ubuntu 22.04 LTS. SSH keys were auto-generated by the CLI at `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub` since no existing key pair was present on the client host.

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

**Actual CLI notice (printed before JSON output):**

```
SSH key files '/root/.ssh/id_rsa' and '/root/.ssh/id_rsa.pub' have been generated under
~/.ssh to allow SSH access to the VM. If using machines without permanent storage, back
up your keys to a safe location.
```

**Actual output:**

```json
{
  "fqdns": "",
  "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-8172d8914a6f4db7/providers/Microsoft.Compute/virtualMachines/xfusion-vm",
  "location": "eastus",
  "macAddress": "60-45-BD-EE-76-46",
  "powerState": "VM running",
  "privateIpAddress": "10.0.0.4",
  "publicIpAddress": "20.124.200.51",
  "resourceGroup": "kml_rg_main-8172d8914a6f4db7",
  "zones": ""
}
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- az vm create full output showing SSH key generation notice and powerState: VM running with publicIpAddress: 20.124.200.51 ]`

---

### Step 6 -- Set the VM Public IP Variable

The public IP address from the VM creation output was captured in a shell variable to be referenced by all subsequent SSH and SCP commands.

```bash
VM_IP="20.124.200.51"
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- VM_IP variable assignment ]`

---

### Step 7 -- Copy the Log Script from Client Host to the VM

The pre-existing `send_logs.py` script at `/root/send_logs.py` on the client host was transferred to the VM's `azureuser` home directory using `scp`. The `StrictHostKeyChecking=no` flag suppressed the interactive host fingerprint prompt on this first connection to the newly provisioned VM.

```bash
scp -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no \
  /root/send_logs.py \
  azureuser@$VM_IP:/home/azureuser/
```

**Actual output:**

```
Warning: Permanently added '20.124.200.51' (ECDSA) to the list of known hosts.
send_logs.py    100%  960     8.2KB/s   00:00
```

The 960-byte script transferred successfully and the host fingerprint was written to `~/.ssh/known_hosts` on the client.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- scp output showing "Warning: Permanently added..." followed by send_logs.py 100% transfer ]`

---

### Step 8 -- Install Python3 pip on the VM

A single SSH command ran `apt-get update` followed by `apt-get install python3-pip` non-interactively on the VM. This pulled 64 new packages and upgraded 5 existing ones, including the entire `build-essential` and `gcc` toolchain as transitive dependencies of the Python development headers.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sudo apt-get update -y && sudo apt-get install -y python3-pip"
```

**Key output metrics from actual run:**

```
Fetched 78.9 MB in 3s (26.8 MB/s)
...
5 upgraded, 64 newly installed, 0 to remove and 52 not upgraded.
Need to get 78.9 MB of archives.
After this operation, 240 MB of additional disk space will be used.
```

**Notable packages installed (from actual output):**

| Package | Role |
|---|---|
| `python3-pip` | Primary target |
| `build-essential` | Compilation toolchain meta-package |
| `gcc-11`, `g++-11` | C/C++ compilers (transitive) |
| `libpython3.10-dev` | Python development headers |
| `python3-wheel` | Wheel build support |

The session also printed a service restart notification, which is expected behaviour after kernel-adjacent package updates on Azure Ubuntu VMs:

```
Services to be restarted:
 systemctl restart walinuxagent.service
```

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- apt-get install completion block showing "64 newly installed" and python3-pip in the package list ]`

---

### Step 9 -- Install Azure SDK Packages on the VM

In a separate SSH command, `pip3 install` was used to install `azure-eventhub` and `azure-storage-blob` along with their dependencies.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "pip3 install azure-eventhub azure-storage-blob"
```

**Actual pip3 output (abbreviated):**

```
Defaulting to user installation because normal site-packages is not writeable
Collecting azure-eventhub
  Downloading azure_eventhub-5.15.1-py3-none-any.whl (317 kB)
Collecting azure-storage-blob
  Downloading azure_storage_blob-12.28.0-py3-none-any.whl (431 kB)
Collecting azure-core>=1.27.0
  Downloading azure_core-1.39.0-py3-none-any.whl (218 kB)
Collecting typing-extensions>=4.0.1
  Downloading typing_extensions-4.15.0-py3-none-any.whl (44 kB)
Requirement already satisfied: cryptography>=2.1.4 in /usr/lib/python3/dist-packages
Collecting isodate>=0.6.1
  Downloading isodate-0.7.2-py3-none-any.whl (22 kB)

Successfully installed azure-core-1.39.0 azure-eventhub-5.15.1 \
  azure-storage-blob-12.28.0 isodate-0.7.2 typing-extensions-4.15.0
```

**Packages installed:**

| Package | Version |
|---|---|
| `azure-eventhub` | 5.15.1 |
| `azure-storage-blob` | 12.28.0 |
| `azure-core` | 1.39.0 |
| `isodate` | 0.7.2 |
| `typing-extensions` | 4.15.0 |

> **Key observation:** pip3 printed `Defaulting to user installation because normal site-packages is not writeable`. The SSH session ran as `azureuser` without `sudo`, so packages were installed to the user scope at `~/.local/lib/`. The install succeeded and the packages were available to `python3` invocations by the same user. See [Lessons Learned](#lessons-learned) for scope implications.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- pip3 install output showing "Defaulting to user installation" warning and "Successfully installed" summary line ]`

---

### Step 10 -- Retrieve Event Hub and Storage Connection Strings

Both connection strings were fetched in a single command block. The Event Hub primary connection string was retrieved from the `RootManageSharedAccessKey` authorization rule of the pre-existing namespace. The Storage connection string was retrieved from the storage account. Both were echoed to stdout to confirm successful retrieval.

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

**Actual output:**

```
EH: Endpoint=sb://xfusion-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=xLb+iF4kknDKjuKgOcNoOlCXbsbSaeLJv+AEhAXIInw=
SA: DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=xfusionst6179;AccountKey=Zwpw/jTPw3wj7xTLCwv7Ppt3T72vX0gS6SeWOSzrXrBRRFYpO8tfKvBrIuDxLR79vAmz6JL1nFl3+AStYpgSNg==;BlobEndpoint=https://xfusionst6179.blob.core.windows.net/;FileEndpoint=https://xfusionst6179.file.core.windows.net/;QueueEndpoint=https://xfusionst6179.queue.core.windows.net/;TableEndpoint=https://xfusionst6179.table.core.windows.net/
```

> **Security Note:** Both strings contain plaintext shared access keys. They were echoed to the terminal for verification purposes only and were rotated immediately after the session. In production, secrets must never be echoed to stdout, stored in shell history, or committed to version control. Use Azure Key Vault with Managed Identity instead.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- echo output showing EH: and SA: connection strings printed to stdout (keys redacted in production documentation) ]`

---

### Step 11 -- Inject Event Hub Connection String into the Script

The `send_logs.py` script on the VM contains the literal placeholder `<Event Hub Connection String>`. This was replaced in-place on the VM using `sed -i` over SSH. The pipe `|` character was used as the `sed` delimiter to safely handle the forward slashes present in the Event Hub endpoint URL.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sed -i 's|<Event Hub Connection String>|$EH_CONN|g' /home/azureuser/send_logs.py"
```

No output is produced on success.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- sed Event Hub injection SSH command completing silently with no error output ]`

---

### Step 12 -- Inject Blob Storage Connection String into the Script

The second placeholder `<Blob Storage Connection String>` was replaced in exactly the same way using a separate SSH command.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "sed -i 's|<Blob Storage Connection String>|$SA_CONN|g' /home/azureuser/send_logs.py"
```

No output is produced on success. After this step the script on the VM contains both live connection strings and is fully configured for execution.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- sed Blob Storage injection SSH command completing silently with no error output ]`

---

### Step 13 -- Execute the Log Sender Script on the VM

The script was executed remotely over a final SSH command. Python3 ran `send_logs.py` on the VM, simultaneously sending a log entry to the Event Hub and writing a backup blob to the storage container.

```bash
ssh -i ~/.ssh/id_rsa -o StrictHostKeyChecking=no azureuser@$VM_IP \
  "python3 /home/azureuser/send_logs.py"
```

**Actual output:**

```
Log sent to Event Hub and backed up to Blob Storage.
```

This single confirmation line indicates both the Event Hub send and the Blob Storage write completed without exception inside the script.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- python3 send_logs.py SSH execution returning the single line: "Log sent to Event Hub and backed up to Blob Storage." ]`

---

### Step 14 -- Verify the Blob Backup in the Container

The blob container was listed using the Azure CLI on the client host (not on the VM) to confirm that `logs.txt` had been written by the script execution in Step 13.

```bash
az storage blob list \
  --container-name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --output table
```

**Actual output:**

```
Name      Blob Type    Blob Tier    Length    Content Type              Last Modified              Snapshot
--------  -----------  -----------  --------  ------------------------  -------------------------  ----------
logs.txt  AppendBlob                18        application/octet-stream  2026-03-27T05:32:41+00:00
```

`logs.txt` was written as an `AppendBlob` of 18 bytes at `05:32:41 UTC`. The `AppendBlob` type confirms the `azure-storage-blob` SDK used the append-optimized client, which is correct for log accumulation workloads where entries are added sequentially without overwriting.

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- az storage blob list table output showing logs.txt as AppendBlob, 18 bytes, last modified 2026-03-27T05:32:41+00:00 ]`

---

### Step 15 -- Validate Event Hub IncomingMessages Metrics

The Event Hub namespace metrics were queried via `az monitor metrics list` using the full ARM resource ID of the `xfusion-namespace` to confirm the message was ingested by the Event Hub.

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-8172d8914a6f4db7/providers/Microsoft.EventHub/namespaces/xfusion-namespace" \
  --metric "IncomingMessages" \
  --output table
```

**Actual output (full table, 60 rows):**

```
Timestamp             Name               Total
--------------------  -----------------  -------
2026-03-27T04:35:00Z  Incoming Messages  0.0
2026-03-27T04:36:00Z  Incoming Messages  0.0
2026-03-27T04:37:00Z  Incoming Messages  0.0
...
2026-03-27T05:32:00Z  Incoming Messages  0.0
2026-03-27T05:33:00Z  Incoming Messages  0.0
2026-03-27T05:34:00Z  Incoming Messages  1.0
```

All 59 preceding minute buckets show `0.0`. At `2026-03-27T05:34:00Z` the metric registers `1.0`, confirming exactly one message was received by the Event Hub. The script executed at `05:32:41 UTC` and the metric reflected this at `05:34:00 UTC`, a metric propagation delay of approximately 79 seconds. This is normal Azure Monitor aggregation behaviour and is addressed in [Lessons Learned](#lessons-learned).

> **Screenshot Placeholder**
> `[ SCREENSHOT: Terminal -- az monitor metrics list full table output showing 59 rows of 0.0 followed by a final row with 1.0 at 2026-03-27T05:34:00Z ]`

---

## Errors Encountered and Resolutions

### Error 1 -- Blob Container Creation Returned `"created": false`

**Occurred at:** Step 2b

**Command:**

```bash
az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
```

**Actual output:**

```json
{ "created": false }
```

**Root Cause:**

Azure storage accounts created via the CLI now have `allowBlobPublicAccess` set to `false` by default as a security hardening measure. When a container creation request includes `--public-access blob` against an account that prohibits public blob access at the account level, Azure rejects the public access assignment silently. The Azure CLI returns `"created": false` with no error message, no warning, and no non-zero exit code. Any automation that relies solely on exit codes to detect failures will miss this entirely.

**Resolution applied:**

Updated the storage account to permit public blob access at the account level, then re-executed the original container creation command:

```bash
az storage account update \
  --name xfusionst6179 \
  --resource-group $RG \
  --allow-blob-public-access true

az storage container create \
  --name xfusion-backup-23374 \
  --account-name xfusionst6179 \
  --account-key $STORAGE_KEY \
  --public-access blob
# Output: { "created": true }
```

**Prevention:**

When public blob access is a genuine requirement, include `--allow-blob-public-access true` directly in the original `az storage account create` command to handle it atomically:

```bash
az storage account create \
  --name xfusionst6179 \
  --resource-group $RG \
  --location eastus \
  --sku Standard_LRS \
  --allow-blob-public-access true
```

**Production Recommendation:**

In production environments, public blob access must remain disabled. Use Shared Access Signatures (SAS tokens), Azure AD RBAC with `Storage Blob Data Contributor` role assignments, or Private Endpoints to grant scoped and authenticated access without opening containers publicly.

---

## Best Practices

### Credential Management

- Never echo connection strings to stdout in production pipelines. The `echo "EH: $EH_CONN"` pattern used in Step 10 is acceptable only for immediate local verification and must not appear in CI/CD logs or persisted output files.
- Store all secrets in **Azure Key Vault** and retrieve them at runtime using `az keyvault secret show` or SDK calls with Managed Identity.
- Prefer **Managed Identity** over shared access keys for all Azure-to-Azure service communication to eliminate key rotation requirements entirely.
- Set `HISTCONTROL=ignorespace` and prefix sensitive commands with a space, or use `unset HISTFILE` for the duration of credential-handling sessions, to prevent connection strings from persisting in `~/.bash_history`.

### Storage Account Hardening

- Always explicitly set `--min-tls-version TLS1_2` at storage account creation time. The provisioned account defaulted to `TLS1_0`, which is deprecated and vulnerable.
- Keep `--allow-blob-public-access false` for production storage accounts. This was the correct default that should have been respected from the start.
- Use **Private Endpoints** or **VNet Service Endpoints** to restrict storage account traffic to specific subnets and eliminate public endpoint exposure.
- Enable **Soft Delete** for blobs (`--enable-blob-delete-retention true`) to protect against accidental deletion during the backup retention window.

### Virtual Machine Access

- Never use `StrictHostKeyChecking=no` in production SSH automation. Pre-populate `~/.ssh/known_hosts` using `ssh-keyscan -H $VM_IP` before the first connection.
- Restrict inbound SSH (TCP/22) to known CIDR ranges using an **NSG inbound rule** on the VM's network interface. The VM as provisioned accepts SSH from `0.0.0.0/0` by default.
- Use **Azure Bastion** for interactive VM access in production environments to remove the requirement for a public IP entirely.
- Back up auto-generated SSH keys to a secrets manager immediately after VM provisioning. The CLI itself emitted the warning to do so.

### Python Dependency Management

- Use `sudo pip3 install` or a dedicated **virtual environment** to ensure packages are accessible system-wide rather than user-scoped.
- Pin all package versions in a `requirements.txt` and deploy via `pip3 install -r requirements.txt` for reproducible environments:

```
azure-eventhub==5.15.1
azure-storage-blob==12.28.0
azure-core==1.39.0
isodate==0.7.2
typing-extensions==4.15.0
```

- Use **Azure Custom Script Extension** or **cloud-init** for idempotent, version-controlled VM bootstrapping rather than sequential, interactive SSH commands.

### Event Hubs Configuration

- Use dedicated **consumer groups** per downstream service to isolate readers and prevent one consumer from blocking another.
- Set **message retention** aligned to downstream consumer processing SLAs (1 to 7 days on Standard tier).
- Enable **Auto-inflate** on the namespace to automatically scale Throughput Units under burst load.
- Monitor `IncomingMessages`, `OutgoingMessages`, and `ThrottledRequests` together to detect backpressure and quota exhaustion proactively.

### Observability and Alerting

- Set an **Azure Monitor Alert** on `IncomingMessages` dropping to zero over a sustained window (e.g., 5 consecutive minutes) to detect pipeline stall conditions automatically.
- Export Event Hub and Storage metrics to a **Log Analytics Workspace** for unified dashboarding, cross-resource correlation, and long-term trend analysis.
- Enable **Azure Storage Diagnostics** logging on the storage account to capture per-blob read and write operations for audit and access pattern analysis.

### Infrastructure as Code

- Convert all `az` CLI commands in this runbook into **Bicep** or **Terraform** templates for repeatable, drift-detectable deployments.
- Store IaC templates in a dedicated repository with branch protection, peer review gates, and automated linting (`az bicep build`, `terraform validate`).
- Apply **Azure Resource Locks** (`CanNotDelete`) to production resource groups to prevent accidental teardown of the pipeline infrastructure.

---

## Lessons Learned

***1. Azure CLI silent failures require JSON output inspection, not just exit codes.***
The container creation failure in Step 2b returned exit code 0 with `"created": false` and zero error output. Any script that relies purely on exit codes to detect `az storage container create` failures will silently continue on this class of error. Always inspect the `created` field in the response JSON, or use `--query "created" -o tsv` to assert the expected value inline in automation.

***2. Storage account public access defaults changed. Design creation commands defensively.***
Azure hardened storage account defaults so that `allowBlobPublicAccess` is `false` at account creation. Any automation that assumes `--public-access blob` will succeed on a newly created account without an explicit account-level flag will fail. When public access is genuinely required, include `--allow-blob-public-access true` in the `az storage account create` command to handle it atomically in a single step instead of discovering the failure in a downstream container operation.

***3. Use the pipe `|` delimiter in `sed` when injecting Azure connection strings.***
Azure connection strings contain forward slashes in endpoint URLs (e.g., `https://xfusion-namespace.servicebus.windows.net/`). Using the default `/` delimiter in `sed 's/.../.../'` would cause sed to misparse the replacement string and fail. The pipe delimiter `sed 's|...|...|g'` was correctly applied here to handle forward slashes transparently. This is a non-obvious but critical detail for any templated credential injection pattern in shell automation.

***4. pip3 installs to user scope when SSH sessions lack elevated privileges.***
The pip3 install produced the warning `Defaulting to user installation because normal site-packages is not writeable` because the SSH session ran as `azureuser` without `sudo`. The install succeeded in user scope at `~/.local/lib/` and worked correctly when `python3 send_logs.py` ran as the same user. However, if the script were to run as a different system user (e.g., via a systemd service or cron job running as `root` or another account), the packages would not be found. Always use `sudo pip3 install` or a virtualenv to ensure the installation scope matches the runtime execution context.

***5. Azure Monitor metric propagation has a 60 to 90 second delay.***
The script executed at `05:32:41 UTC` and the `IncomingMessages` metric registered `1.0` only at the `05:34:00 UTC` bucket, a lag of approximately 79 seconds. Running `az monitor metrics list` immediately after script execution would show `0.0` and could be mistaken for a pipeline failure. Build any metric-based verification step with an explicit wait interval (at least 90 seconds) or a poll loop with retries rather than a single point-in-time check.

***6. AppendBlob type in the blob list output confirms correct SDK usage.***
The blob `logs.txt` was stored as an `AppendBlob` (confirmed in Step 14). This is the appropriate blob type for append-only log writes. If the script had used `BlockBlobClient` instead, each execution would have overwritten the file entirely, destroying prior log entries. The `AppendBlob` result confirms the SDK used `AppendBlobClient` inside `send_logs.py`, preserving historical log entries across successive script runs.

***7. Default `minimumTlsVersion` is `TLS1_0` and must be explicitly remediated.***
The Step 1 output clearly showed `"minimumTlsVersion": "TLS1_0"` on the newly created storage account. TLS 1.0 is deprecated and vulnerable to known downgrade attacks. This must be flagged in any post-deployment security review and remediated immediately:

```bash
az storage account update \
  --name xfusionst6179 \
  --resource-group $RG \
  --min-tls-version TLS1_2
```

In future, include `--min-tls-version TLS1_2` in the original `az storage account create` command.

***8. Auto-generated SSH keys must be backed up before the session ends.***
The `az vm create --generate-ssh-keys` flag created `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub` on the client host. The CLI explicitly warned that these must be backed up if the host does not use permanent storage. In any ephemeral or container-based environment (a CI runner, a cloud shell session, a disposable bastion), failure to back up the private key before the session terminates permanently locks out SSH access to the VM without a key reset operation.

---

## Resource Reference

| Resource Type | Name | Detail |
|---|---|---|
| Subscription | -- | `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Resource Group | `kml_rg_main-8172d8914a6f4db7` | East US |
| Event Hubs Namespace | `xfusion-namespace` | Standard tier, East US |
| Storage Account | `xfusionst6179` | Standard_LRS, StorageV2, East US |
| Blob Container | `xfusion-backup-23374` | Public access: blob |
| Blob Object | `logs.txt` | AppendBlob, 18 bytes, `application/octet-stream` |
| Virtual Machine | `xfusion-vm` | Ubuntu 22.04 LTS, Standard_B1s, East US |
| VM OS Disk | -- | 64 GB, Standard_LRS |
| VM Admin User | `azureuser` | Key-based SSH auth |
| VM Public IP | `20.124.200.51` | Dynamic, East US |
| VM Private IP | `10.0.0.4` | Within VNet |
| VM MAC Address | `60-45-BD-EE-76-46` | -- |
| Client Script (source) | `/root/send_logs.py` | 960 bytes, client host |
| Script (deployed) | `/home/azureuser/send_logs.py` | On VM after SCP in Step 7 |
| SSH Private Key | `~/.ssh/id_rsa` | Auto-generated during Step 5 |

---

## Author

**Nautilus DevOps Team**
Runbook executed: `2026-03-27`
Execution environment: Azure Cloud -- East US
Authentication: Azure CLI (`az`) with pre-authenticated subscription session
Toolchain: Azure CLI, SSH, SCP, Python 3.10, pip3, sed
**All infrastructure provisioned exclusively via Azure CLI. No Azure Portal interactions were performed at any step.**

---

*All credentials and connection strings visible in terminal outputs in this document were rotated immediately after execution. Do not commit live secrets to any version control system. All `$STORAGE_KEY`, `$EH_CONN`, and `$SA_CONN` values are ephemeral shell variables that do not persist beyond the session.*





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
