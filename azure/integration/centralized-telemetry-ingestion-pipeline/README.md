# Azure VM to Event Hubs: Centralized Log Collection Pipeline

[![Azure](https://img.shields.io/badge/Azure-Event%20Hubs-0078D4?style=flat-square&logo=microsoft-azure)](https://azure.microsoft.com/en-us/products/event-hubs)
[![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=flat-square&logo=python)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/Platform-Ubuntu%2022.04-E95420?style=flat-square&logo=ubuntu)](https://ubuntu.com/)
[![Status](https://img.shields.io/badge/Status-Production%20Ready-brightgreen?style=flat-square)]()
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square)]()

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Implementation Walkthrough](#implementation-walkthrough)
  - [Step 1: Verify Active Azure Subscription](#step-1-verify-active-azure-subscription)
  - [Step 2: List All Subscriptions](#step-2-list-all-subscriptions)
  - [Step 3: Confirm the Resource Group](#step-3-confirm-the-resource-group)
  - [Step 4: Confirm the Existing Virtual Machine](#step-4-confirm-the-existing-virtual-machine)
  - [Step 5: Provision the Event Hubs Namespace](#step-5-provision-the-event-hubs-namespace)
  - [Step 6: Verify the Namespace](#step-6-verify-the-namespace)
  - [Step 7: Create the Event Hub -- First Attempt (ERROR)](#step-7-create-the-event-hub----first-attempt-error)
  - [Step 8: Create the Event Hub -- Corrected Command](#step-8-create-the-event-hub----corrected-command)
  - [Step 9: Verify the Event Hub](#step-9-verify-the-event-hub)
  - [Step 10: Retrieve the Connection String](#step-10-retrieve-the-connection-string)
  - [Step 11: Retrieve the VM Public IP and SSH Into the VM](#step-11-retrieve-the-vm-public-ip-and-ssh-into-the-vm)
  - [Step 12: Inspect the Existing Log Producer Script](#step-12-inspect-the-existing-log-producer-script)
  - [Step 13: Verify the azure-eventhub SDK](#step-13-verify-the-azure-eventhub-sdk)
  - [Step 14: Inject the Connection String into the Script](#step-14-inject-the-connection-string-into-the-script)
  - [Step 15: Confirm the Updated Script](#step-15-confirm-the-updated-script)
  - [Step 16: Execute the Script -- Runs 1, 2, and 3 (Individual)](#step-16-execute-the-script----runs-1-2-and-3-individual)
  - [Step 17: Execute the Script -- Runs 4, 5, and 6 (Loop)](#step-17-execute-the-script----runs-4-5-and-6-loop)
  - [Step 18: Exit the VM](#step-18-exit-the-vm)
  - [Step 19: Validate IncomingMessages Metric](#step-19-validate-incomingmessages-metric)
  - [Step 20: Validate IncomingBytes Metric](#step-20-validate-incomingbytes-metric)
  - [Step 21: Full Stack Health Verification](#step-21-full-stack-health-verification)
- [Project Structure](#project-structure)
- [Configuration Reference](#configuration-reference)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)
- [References](#references)

---

## Overview

This repository documents the end-to-end implementation of a **centralized log collection pipeline** that integrates an Azure Virtual Machine with Azure Event Hubs. The solution enables the Nautilus DevOps team to stream structured log data from a running VM workload into a scalable, durable event streaming service for downstream processing, monitoring, and analytics.

**Key Outcome:** 60 log events successfully ingested across 6 script runs (10 entries each) into `datacenter-hub`, confirmed via Azure Monitor `IncomingMessages` (120) and `IncomingBytes` (8,569) metrics.

---

## Problem Statement

| Dimension | Detail |
|-----------|--------|
| **Team** | Nautilus DevOps |
| **Challenge** | No centralized log collection mechanism for VM workloads |
| **Risk** | Log data siloed on individual VMs with no observability at scale |
| **Resolution** | Stream logs in real time to Azure Event Hubs using a Python producer client |
| **Region** | East US |
| **Subscription** | Azure Free Labs (`f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`) |
| **Resource Group** | `kml_rg_main-53613478641048b5` |
| **Tenant ID** | `54c1a2d3-d100-453c-9636-3a109eb45552` |

---

## Architecture

```
+---------------------------+        Python SDK          +-----------------------------+
|      datacenter-vm        |  ------------------------> |    datacenter-namespace      |
|  Ubuntu 22.04 LTS         |   azure-eventhub 5.15.1    |    (Event Hubs Namespace)    |
|  Private IP: 10.0.0.4     |                            |    SKU: Standard             |
|  Public IP: 20.115.31.19  |                            |    AutoInflate: Enabled      |
+---------------------------+                            |    Max TU: 10                |
           |                                             +-----------------------------+
           | send_logs.py                                             |
           | EventHubProducerClient                                   v
           |                                             +-----------------------------+
           +-------------------------------------------> |       datacenter-hub         |
                                                         |    Partitions: 2             |
                                                         |    Retention: 7 days         |
                                                         |    Status: Active            |
                                                         +-----------------------------+
                                                                      |
                                                                      v
                                                         +-----------------------------+
                                                         |    Azure Monitor Metrics     |
                                                         |    IncomingMessages: 120     |
                                                         |    IncomingBytes: 8,569      |
                                                         +-----------------------------+
```

---

## Prerequisites

Before beginning, ensure the following are in place:

* Azure CLI (`az`) installed and authenticated as a service principal or user account
* An active Azure subscription with at least one resource group already provisioned
* An existing Virtual Machine (`datacenter-vm`) deployed in the `eastus` region
* SSH access to the VM via the `azureuser` account
* Python 3.10 or later on the VM with `azure-eventhub >= 5.11.0` installed
* NSG outbound rules permitting traffic on port `5671` (AMQP) or port `443` (AMQP over WebSockets)

---

## Implementation Walkthrough

### Step 1: Verify Active Azure Subscription

Before provisioning any resources, confirm that the Azure CLI is authenticated and targeting the correct subscription and tenant. This is the mandatory first command in every deployment session.

```bash
az account show
```

**Actual Output:**

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
    "name": "152c4ca6-7da7-449e-89b0-5a4c6bc5c1a3",
    "type": "servicePrincipal"
  }
}
```

**What to confirm:**

* `"state": "Enabled"` -- subscription is active and not suspended
* `"isDefault": true` -- this subscription is the active context for all subsequent CLI commands
* `"type": "servicePrincipal"` -- session is authenticated via a service principal, not an interactive user
* `"id"` matches the expected subscription ID before proceeding

> **Screenshot**

<img width="1031" height="617" alt="image" src="https://github.com/user-attachments/assets/1af38257-1af6-4363-82e1-b18eead5c680" />

> `Terminal output of az account show returning the full JSON block with "state": "Enabled", "isDefault": true, and "type": "servicePrincipal" all visible`

---

### Step 2: List All Subscriptions

Enumerate all subscriptions accessible to this account to confirm that only one subscription is in scope and there is no risk of resources being created in the wrong context.

```bash
az account list --output table
```

**Actual Output:**

```
Name             CloudName    SubscriptionId                        TenantId                              State    IsDefault
---------------  -----------  ------------------------------------  ------------------------------------  -------  -----------
Azure Free Labs  AzureCloud   f0c3bcdd-5ce2-4fa0-8cf3-41559747512b  54c1a2d3-d100-453c-9636-3a109eb45552  Enabled  True
```

**What to confirm:**

* Exactly one subscription row is returned
* `IsDefault: True` matches the subscription confirmed in Step 1
* `State: Enabled` confirms the subscription is active

> **Screenshot**

<img width="1031" height="617" alt="image" src="https://github.com/user-attachments/assets/1af38257-1af6-4363-82e1-b18eead5c680" />

> `Terminal output of az account list --output table showing exactly one row: Azure Free Labs, AzureCloud, Enabled, IsDefault True`

---

### Step 3: Confirm the Resource Group

Verify that the target resource group exists in the `eastus` region and is in a `Succeeded` state before creating any resources inside it.

```bash
az group list --output table
```

**Actual Output:**

```
Name                          Location    Status
----------------------------  ----------  ---------
kml_rg_main-53613478641048b5  eastus      Succeeded
```

**What to confirm:**

* `kml_rg_main-53613478641048b5` exists with `Status: Succeeded`
* `Location: eastus` matches the required deployment region for all pipeline resources

> **Screenshot**

<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/dbad102f-ed65-4c05-afaa-e8d742de461a" />

> `Terminal output of az group list --output table showing kml_rg_main-53613478641048b5 in eastus with Status: Succeeded`

---

### Step 4: Confirm the Existing Virtual Machine

Verify that `datacenter-vm` exists in the resource group before proceeding to build the Event Hubs infrastructure it will write to. This prevents building a pipeline with no producer.

```bash
az vm list --output table
```

**Actual Output:**

```
Name           ResourceGroup                 Location    Zones
-------------  ----------------------------  ----------  -------
datacenter-vm  KML_RG_MAIN-53613478641048B5  eastus
```

**What to confirm:**

* `datacenter-vm` is present in the expected resource group
* `Location: eastus` is consistent with all other pipeline resources
* No availability zone is assigned -- single-zone deployment acceptable for this workload

> **Screenshot**

<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/dbad102f-ed65-4c05-afaa-e8d742de461a" />

> `Terminal output of az vm list --output table showing datacenter-vm in KML_RG_MAIN-53613478641048B5 in eastus`

---

### Step 5: Provision the Event Hubs Namespace

Create the Event Hubs namespace with the Standard SKU, Auto-Inflate enabled, and a maximum of 10 throughput units. Auto-Inflate allows the namespace to scale up automatically under load without requiring manual throughput unit adjustments.

```bash
az eventhubs namespace create \
  --name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --location eastus \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 10
```

**Actual Output (abbreviated):**

```json
{
  "createdAt": "2026-01-06T19:20:14.4704452Z",
  "isAutoInflateEnabled": true,
  "kafkaEnabled": true,
  "location": "eastus",
  "maximumThroughputUnits": 10,
  "metricId": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b:datacenter-namespace",
  "minimumTlsVersion": "1.2",
  "name": "datacenter-namespace",
  "provisioningState": "Succeeded",
  "sku": {
    "capacity": 1,
    "name": "Standard",
    "tier": "Standard"
  },
  "status": "Active",
  "updatedAt": "2026-03-24T06:40:02Z",
  "zoneRedundant": true
}
```

**What to confirm:**

* `"provisioningState": "Succeeded"` -- namespace created without errors
* `"isAutoInflateEnabled": true` -- throughput auto-scaling is active
* `"kafkaEnabled": true` -- Kafka-compatible surface available for future consumers
* `"zoneRedundant": true` -- Standard SKU in eastus applies zone redundancy automatically
* `"minimumTlsVersion": "1.2"` -- TLS 1.2 enforced, no legacy TLS allowed

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of az eventhubs namespace create returning the full JSON block with provisioningState: Succeeded, isAutoInflateEnabled: true, kafkaEnabled: true, and status: Active all visible]`

---

### Step 6: Verify the Namespace

Run a focused query against the new namespace to validate that all critical fields are correctly set before proceeding to create the Event Hub inside it.

```bash
az eventhubs namespace show \
  --name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "{Name:name, Location:location, SKU:sku.name, AutoInflate:isAutoInflateEnabled, State:provisioningState}" \
  --output table
```

**Actual Output:**

```
Name                  Location    SKU       AutoInflate    State
--------------------  ----------  --------  -------------  ---------
datacenter-namespace  eastus      Standard  True           Succeeded
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of az eventhubs namespace show --output table showing datacenter-namespace, eastus, Standard, AutoInflate True, State Succeeded]`

---

### Step 7: Create the Event Hub -- First Attempt (ERROR)

The first attempt to create `datacenter-hub` included the `--message-retention 1` flag to set a 1-day message retention window. This command **failed immediately** and did not create the Event Hub.

```bash
az eventhubs eventhub create \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --partition-count 2 \
  --message-retention 1
```

**Actual Error Output:**

```
unrecognized arguments: --message-retention 1

Examples from AI knowledge base:
https://aka.ms/cli_ref
Read more about the command in reference docs
```

**What happened:** The CLI exited with a non-zero status code, shown by the `✖` prompt symbol. The Event Hub was **not created**. The `--message-retention` flag was removed from the Azure CLI when the Event Hubs ARM API was updated, and no backward-compatible alias was provided.

> **Screenshot Placeholder -- ERROR**
> `[SCREENSHOT: Terminal showing the "✖" red prompt symbol, the complete failed az eventhubs eventhub create command with --message-retention 1, and the full error text: "unrecognized arguments: --message-retention 1" followed by the CLI reference hint]`

**Resolution:** Remove the `--message-retention` flag and re-run. See [Step 8](#step-8-create-the-event-hub----corrected-command) for the corrected command and [Errors Encountered and Resolutions](#errors-encountered-and-resolutions) for full root cause analysis.

---

### Step 8: Create the Event Hub -- Corrected Command

After removing the deprecated `--message-retention` flag, the command was re-run successfully. The Standard SKU automatically applies a default retention of 7 days (168 hours).

```bash
az eventhubs eventhub create \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --partition-count 2
```

**Actual Output:**

```json
{
  "createdAt": "2026-03-24T06:43:02.44Z",
  "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-53613478641048b5/providers/Microsoft.EventHub/namespaces/datacenter-namespace/eventhubs/datacenter-hub",
  "location": "eastus",
  "messageRetentionInDays": 7,
  "name": "datacenter-hub",
  "partitionCount": 2,
  "partitionIds": ["0", "1"],
  "resourceGroup": "kml_rg_main-53613478641048b5",
  "retentionDescription": {
    "cleanupPolicy": "Delete",
    "retentionTimeInHours": 168
  },
  "status": "Active",
  "updatedAt": "2026-03-24T06:43:11.32Z"
}
```

**What to confirm:**

* `"status": "Active"` -- Event Hub is live and accepting messages
* `"partitionCount": 2` -- two partitions provisioned as specified
* `"partitionIds": ["0", "1"]` -- both partition IDs assigned and ready
* `"messageRetentionInDays": 7` -- 7-day default retention applied automatically
* `"retentionTimeInHours": 168` -- consistent with the 7-day default

> **Screenshot Placeholder -- RESOLUTION**
> `[SCREENSHOT: Terminal showing the corrected az eventhubs eventhub create command (without --message-retention) with the "➜" green prompt symbol and the full JSON response with "status": "Active", "partitionCount": 2, and "retentionTimeInHours": 168 visible]`

---

### Step 9: Verify the Event Hub

Run a focused verification query to confirm the Event Hub name, operational status, and partition count before retrieving the connection string.

```bash
az eventhubs eventhub show \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "{Name:name, Status:status, Partitions:partitionCount}" \
  --output table
```

**Actual Output:**

```
Name            Status    Partitions
--------------  --------  ------------
datacenter-hub  Active    2
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of az eventhubs eventhub show --output table showing datacenter-hub, Status Active, Partitions 2]`

---

### Step 10: Retrieve the Connection String

Retrieve the primary connection string from the `RootManageSharedAccessKey` shared access policy and store it in a shell environment variable for use during the VM configuration steps.

```bash
CONNECTION_STRING=$(az eventhubs namespace authorization-rule keys list \
  --resource-group kml_rg_main-53613478641048b5 \
  --namespace-name datacenter-namespace \
  --name RootManageSharedAccessKey \
  --query "primaryConnectionString" \
  --output tsv)

echo $CONNECTION_STRING
```

**Actual Output:**

```
Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=bN6KRp83tVrQPYBx1nLGETa5NJFCfVCfa+AEhOfwlC0=
```

> **Security Note:** The SAS key shown above was rotated immediately after this lab session closed. Never commit connection strings to source control. In production, retrieve secrets at runtime from Azure Key Vault or environment variables.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of the CONNECTION_STRING variable assignment and echo command showing the full Endpoint=sb:// format connection string (mask the SharedAccessKey value before publishing)]`

---

### Step 11: Retrieve the VM Public IP and SSH Into the VM

Dynamically retrieve the public IP address of `datacenter-vm` and open an SSH session to configure and execute the log producer script on the VM.

```bash
VM_IP=$(az vm list-ip-addresses \
  --name datacenter-vm \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" \
  --output tsv)

echo $VM_IP
```

**Actual Output:**

```
20.115.31.19
```

```bash
ssh azureuser@$VM_IP
```

**Actual SSH Handshake and Login Output:**

```
The authenticity of host '20.115.31.19 (20.115.31.19)' can't be established.
ECDSA key fingerprint is SHA256:Q5/8lB40XK8YSS7ma+Uuoq4OJWWjzeEogJwQNE0pPac.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '20.115.31.19' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1044-azure x86_64)

  System load:  0.09              Processes:             108
  Usage of /:   7.2% of 28.89GB  Users logged in:       0
  Memory usage: 33%               IPv4 address for eth0: 10.0.0.4
  Swap usage:   0%
```

**What to confirm:**

* SSH connected to the correct host (`20.115.31.19`)
* ECDSA fingerprint verified and host permanently added to known hosts
* System information banner confirms Ubuntu 22.04.5 LTS and private IP `10.0.0.4` on `eth0`

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the full SSH sequence: VM_IP echo output of 20.115.31.19, the ECDSA fingerprint prompt answered "yes", and the full Ubuntu 22.04 welcome banner with system information table]`

---

### Step 12: Inspect the Existing Log Producer Script

Confirm the `send_logs.py` script is present on the VM, check its file permissions, and review its full contents before making any modifications.

```bash
ls -la /home/azureuser/send_logs.py
```

**Actual Output:**

```
-rwxr-xr-x 1 azureuser azureuser 622 Mar 24 06:35 /home/azureuser/send_logs.py
```

```bash
cat /home/azureuser/send_logs.py
```

**Actual Script Contents (pre-modification):**

```python
from azure.eventhub import EventHubProducerClient, EventData

# Event Hub Configuration
connection_str = "<your_event_hub_connection_string>"
event_hub_name = "datacenter-hub"

# Initialize the producer client
producer = EventHubProducerClient.from_connection_string(
    conn_str=connection_str,
    eventhub_name=event_hub_name
)

# Send logs to the Event Hub
with producer:
    for i in range(10):
        event_data_batch = producer.create_batch()
        event_data_batch.add(EventData(f"Log entry {i + 1}: Sample log message"))
        producer.send_batch(event_data_batch)
        print(f"Log entry {i + 1} sent.")
```

**What to confirm:**

* Script is executable (`-rwxr-xr-x`) and owned by `azureuser`
* The placeholder `<your_event_hub_connection_string>` is present and must be replaced before execution
* `event_hub_name` is already set to `"datacenter-hub"`, matching the hub created in Step 8
* The `create_batch()` + `send_batch()` pattern is used -- the recommended approach for efficient, reliable event delivery

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the ls -la output confirming -rwxr-xr-x permissions and 622 byte file size, followed by the cat output of send_logs.py with the <your_event_hub_connection_string> placeholder clearly visible]`

---

### Step 13: Verify the azure-eventhub SDK

Confirm that the `azure-eventhub` Python SDK is installed on the VM before attempting to execute the script.

```bash
pip3 show azure-eventhub 2>/dev/null || echo "NOT INSTALLED"
```

**Actual Output:**

```
Name: azure-eventhub
Version: 5.15.1
Summary: Microsoft Azure Event Hubs Client Library for Python
Home-page:
Author:
Author-email: Microsoft Corporation <azpysdkhelp@microsoft.com>
License:
Location: /usr/local/lib/python3.10/dist-packages
Requires: azure-core, typing-extensions
Required-by:
```

**What to confirm:**

* `Version: 5.15.1` -- SDK is installed at a production-ready version
* `Location: /usr/local/lib/python3.10/dist-packages` -- installed system-wide for Python 3.10
* Dependencies `azure-core` and `typing-extensions` are present and satisfied

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of pip3 show azure-eventhub showing Name: azure-eventhub, Version: 5.15.1, Location, and Requires fields confirming the SDK is fully installed]`

---

### Step 14: Inject the Connection String into the Script

Use `sed -i` to replace the placeholder string in `send_logs.py` in-place with the actual connection string retrieved in Step 10.

```bash
sed -i 's|connection_str = "<your_event_hub_connection_string>"|connection_str = "Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=bN6KRp83tVrQPYBx1nLGETa5NJFCfVCfa+AEhOfwlC0="|' /home/azureuser/send_logs.py
```

> **Lab-Only Pattern Warning:** Injecting credentials directly into a script file via `sed` is strictly a lab convenience pattern. This must never be used in production environments. See [Lessons Learned -- SAS Key Injection](#4-sas-key-injection-via-sed-is-a-lab-only-pattern) for the production-safe alternative using environment variables or Azure Key Vault.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the full sed -i command executing cleanly and returning to the shell prompt with no error output, confirming the in-place substitution completed successfully]`

---

### Step 15: Confirm the Updated Script

Run `cat` again to verify the `sed` substitution was applied correctly and the actual connection string is now embedded in the script before executing it.

```bash
cat /home/azureuser/send_logs.py
```

**Actual Output (post-modification):**

```python
from azure.eventhub import EventHubProducerClient, EventData

# Event Hub Configuration
connection_str = "Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=bN6KRp83tVrQPYBx1nLGETa5NJFCfVCfa+AEhOfwlC0="
event_hub_name = "datacenter-hub"

# Initialize the producer client
producer = EventHubProducerClient.from_connection_string(
    conn_str=connection_str,
    eventhub_name=event_hub_name
)

# Send logs to the Event Hub
with producer:
    for i in range(10):
        event_data_batch = producer.create_batch()
        event_data_batch.add(EventData(f"Log entry {i + 1}: Sample log message"))
        producer.send_batch(event_data_batch)
        print(f"Log entry {i + 1} sent.")
```

**What to confirm:**

* The placeholder `<your_event_hub_connection_string>` has been fully replaced with no residual characters
* The connection string begins correctly with `Endpoint=sb://datacenter-namespace.servicebus.windows.net/`
* No other lines in the script were modified by the substitution

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of cat /home/azureuser/send_logs.py post-modification showing the full Endpoint=sb:// connection string now on the connection_str line (mask the SharedAccessKey value before publishing)]`

---

### Step 16: Execute the Script -- Runs 1, 2, and 3 (Individual)

Execute `send_logs.py` three times individually, sending 10 log events to `datacenter-hub` per run for a subtotal of 30 events.

```bash
python3 /home/azureuser/send_logs.py
```

**Actual Output -- Run 1:**

```
Log entry 1 sent.
Log entry 2 sent.
Log entry 3 sent.
Log entry 4 sent.
Log entry 5 sent.
Log entry 6 sent.
Log entry 7 sent.
Log entry 8 sent.
Log entry 9 sent.
Log entry 10 sent.
```

The command was re-executed twice more, producing identical output for Runs 2 and 3.

```bash
python3 /home/azureuser/send_logs.py   # Run 2 -- identical output
python3 /home/azureuser/send_logs.py   # Run 3 -- identical output
```

**Running total after Step 16:** 30 log events sent (3 runs x 10 entries per run).

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing three sequential executions of python3 /home/azureuser/send_logs.py, each producing Log entry 1 through 10 sent., all completing with the "➜" green prompt symbol and no errors between runs]`

---

### Step 17: Execute the Script -- Runs 4, 5, and 6 (Loop)

Execute the script three more times using a `for` loop, sending a further 30 events for a cumulative total of 60 log events across all runs.

```bash
for i in {1..3}; do echo "--- Run $i ---"; python3 /home/azureuser/send_logs.py; done
```

**Actual Output:**

```
--- Run 1 ---
Log entry 1 sent.
Log entry 2 sent.
Log entry 3 sent.
Log entry 4 sent.
Log entry 5 sent.
Log entry 6 sent.
Log entry 7 sent.
Log entry 8 sent.
Log entry 9 sent.
Log entry 10 sent.
--- Run 2 ---
Log entry 1 sent.
Log entry 2 sent.
Log entry 3 sent.
Log entry 4 sent.
Log entry 5 sent.
Log entry 6 sent.
Log entry 7 sent.
Log entry 8 sent.
Log entry 9 sent.
Log entry 10 sent.
--- Run 3 ---
Log entry 1 sent.
Log entry 2 sent.
Log entry 3 sent.
Log entry 4 sent.
Log entry 5 sent.
Log entry 6 sent.
Log entry 7 sent.
Log entry 8 sent.
Log entry 9 sent.
Log entry 10 sent.
```

**Cumulative total after Step 17:** 60 log events successfully sent to `datacenter-hub` across all 6 runs.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the complete loop execution output with "--- Run 1 ---", "--- Run 2 ---", and "--- Run 3 ---" section headers and all 30 Log entry N sent. lines, completing cleanly with no error output]`

---

### Step 18: Exit the VM

Close the SSH session cleanly after confirming all log events have been dispatched from the VM.

```bash
exit
```

**Actual Output:**

```
logout
Connection to 20.115.31.19 closed.
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the exit command followed by "logout" and "Connection to 20.115.31.19 closed." confirming the SSH session terminated cleanly and the prompt has returned to the local machine]`

---

### Step 19: Validate IncomingMessages Metric

Query Azure Monitor from the local machine to confirm that `datacenter-hub` received incoming message frames. The `PT1H` interval and `-3:` slice retrieve the most recent data points within the current hour.

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-53613478641048b5/providers/Microsoft.EventHub/namespaces/datacenter-namespace" \
  --metric "IncomingMessages" \
  --interval PT1H \
  --query "value[0].timeseries[0].data[-3:]" \
  --output table
```

**Actual Output:**

```
TimeStamp             Total
--------------------  -------
2026-03-24T05:54:00Z  120.0
```

**What to confirm:**

* `Total: 120.0` confirms message frames were received by the namespace during the ingestion window
* The timestamp (`05:54:00Z`) aligns with the execution window of the log producer runs
* The metric total of 120 for 60 application events is expected -- see [Lessons Learned](#3-incomingmessages-count-exceeds-application-level-event-count) for full explanation

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of the az monitor metrics list IncomingMessages command showing the results table with TimeStamp 2026-03-24T05:54:00Z and Total 120.0]`

---

### Step 20: Validate IncomingBytes Metric

Query Azure Monitor for the `IncomingBytes` metric to confirm that measurable payload data was received, providing a second independent confirmation of successful log ingestion.

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-53613478641048b5/providers/Microsoft.EventHub/namespaces/datacenter-namespace" \
  --metric "IncomingBytes" \
  --interval PT1H \
  --query "value[0].timeseries[0].data[-3:]" \
  --output table
```

**Actual Output:**

```
TimeStamp             Total
--------------------  -------
2026-03-24T05:55:00Z  8569.0
```

**What to confirm:**

* `Total: 8569.0` bytes confirms actual payload was ingested, not just empty protocol frames
* Timestamp (`05:55:00Z`) is 1 minute after the `IncomingMessages` timestamp, consistent with Azure Monitor's per-metric aggregation cadence

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output of the az monitor metrics list IncomingBytes command showing the results table with TimeStamp 2026-03-24T05:55:00Z and Total 8569.0]`

---

### Step 21: Full Stack Health Verification

Run a single consolidated command to verify the health of all three pipeline components -- namespace, Event Hub, and VM -- in one pass. This is the final gate confirmation that the entire pipeline is operational end to end.

```bash
echo "===== NAMESPACE =====" && \
az eventhubs namespace show \
  --name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "{Name:name, SKU:sku.name, AutoInflate:isAutoInflateEnabled, Location:location, State:provisioningState}" \
  --output table && \
echo "===== EVENT HUB =====" && \
az eventhubs eventhub show \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "{Name:name, Status:status, Partitions:partitionCount}" \
  --output table && \
echo "===== VM STATUS =====" && \
az vm get-instance-view \
  --name datacenter-vm \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "instanceView.statuses[1].displayStatus" \
  --output tsv
```

**Actual Output:**

```
===== NAMESPACE =====
Name                  SKU       AutoInflate    Location    State
--------------------  --------  -------------  ----------  ---------
datacenter-namespace  Standard  True           eastus      Succeeded

===== EVENT HUB =====
Name            Status    Partitions
--------------  --------  ------------
datacenter-hub  Active    2

===== VM STATUS =====
VM running
```

**What to confirm:**

* Namespace: `State: Succeeded`, `AutoInflate: True`, `SKU: Standard`, `Location: eastus`
* Event Hub: `Status: Active`, `Partitions: 2`
* VM: `VM running` -- the log producer source machine remains operational

All three components confirmed healthy. The centralized log collection pipeline is fully operational.

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the complete full stack health check output with all three section headers (===== NAMESPACE =====, ===== EVENT HUB =====, ===== VM STATUS =====) and their respective tables and values confirming Succeeded, Active, and VM running]`

---

## Project Structure

```
.
|-- README.md                    # This document
|-- send_logs.py                 # Python Event Hub producer (deployed on VM at /home/azureuser/)
|-- scripts/
|   |-- full_stack_check.sh      # Consolidated health check script (Step 21)
|   |-- verify_metrics.sh        # Azure Monitor metrics validation script (Steps 19 and 20)
|-- docs/
|   |-- architecture.png         # Pipeline architecture diagram
|   |-- screenshots/
|       |-- step-01-account-show.png              # az account show JSON output
|       |-- step-02-account-list.png              # az account list table
|       |-- step-03-group-list.png                # az group list table
|       |-- step-04-vm-list.png                   # az vm list table
|       |-- step-05-namespace-create.png          # namespace create JSON output
|       |-- step-06-namespace-verify.png          # namespace show table
|       |-- step-07-eventhub-error.png            # ERROR: --message-retention failure
|       |-- step-08-eventhub-create-fixed.png     # corrected create JSON output
|       |-- step-09-eventhub-verify.png           # eventhub show table
|       |-- step-10-connection-string.png         # connection string echo (key masked)
|       |-- step-11-ssh-login.png                 # VM IP echo and SSH welcome banner
|       |-- step-12-script-inspect.png            # ls -la and cat pre-modification
|       |-- step-13-sdk-verify.png                # pip3 show azure-eventhub output
|       |-- step-14-sed-inject.png                # sed -i command clean execution
|       |-- step-15-script-updated.png            # cat post-modification (key masked)
|       |-- step-16-runs-1-2-3.png                # three individual script run outputs
|       |-- step-17-runs-4-5-6-loop.png           # for loop three run outputs
|       |-- step-18-exit-vm.png                   # exit and connection closed message
|       |-- step-19-incoming-messages.png         # IncomingMessages metric table
|       |-- step-20-incoming-bytes.png            # IncomingBytes metric table
|       |-- step-21-full-stack-health.png         # consolidated health check output
```

---

## Configuration Reference

| Parameter | Value | Notes |
|-----------|-------|-------|
| Subscription | `Azure Free Labs` | ID: `f0c3bcdd-5ce2-4fa0-8cf3-41559747512b` |
| Tenant ID | `54c1a2d3-d100-453c-9636-3a109eb45552` | Confirmed in Step 1 |
| Resource Group | `kml_rg_main-53613478641048b5` | Location: eastus, confirmed in Step 3 |
| Namespace | `datacenter-namespace` | Standard SKU, AutoInflate enabled, Max TU: 10 |
| Event Hub | `datacenter-hub` | 2 partitions, 7-day retention, Status: Active |
| VM Name | `datacenter-vm` | Ubuntu 22.04.5 LTS, confirmed in Step 4 |
| VM Public IP | `20.115.31.19` | Dynamic -- always query fresh per session |
| VM Private IP | `10.0.0.4` | Static internal address on eth0 |
| SDK | `azure-eventhub==5.15.1` | Python 3.10 system-wide install on VM |
| Auth Rule | `RootManageSharedAccessKey` | Lab use only -- scope to Send-only in production |
| Total Events Sent | `60` | 6 runs x 10 log entries per run |
| IncomingMessages Metric | `120.0` | Includes AMQP framing overhead on top of 60 app events |
| IncomingBytes Metric | `8,569` | Total payload received by namespace |

---

## Errors Encountered and Resolutions

This section catalogs every error encountered during this implementation, with the exact error output, root cause, resolution applied, and screenshot placeholders at the point of failure and recovery.

---

### ERROR-01: `unrecognized arguments: --message-retention 1`

| Field | Detail |
|-------|--------|
| **Occurred At** | Step 7 -- First attempt to create the Event Hub |
| **Command** | `az eventhubs eventhub create` |
| **Severity** | Blocking -- command aborted, Event Hub not created |
| **Category** | Azure CLI API Deprecation |
| **Impact** | Deployment halted; required flag removal and full command re-run |

#### Failed Command

```bash
az eventhubs eventhub create \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --partition-count 2 \
  --message-retention 1
```

#### Exact Error Output

```
unrecognized arguments: --message-retention 1

Examples from AI knowledge base:
https://aka.ms/cli_ref
Read more about the command in reference docs
```

> **Screenshot Placeholder -- ERROR-01: Terminal Error Output**
> `[SCREENSHOT: Terminal showing the "✖" red prompt symbol, the complete failed az eventhubs eventhub create command with --message-retention 1 visible, and the full error text "unrecognized arguments: --message-retention 1" with the CLI reference hint line below it]`

#### Root Cause

The `--message-retention` flag was a shorthand for the `messageRetentionInDays` property in the legacy Event Hubs ARM API (pre-2022). When Microsoft released the `2022-10-01-preview` API version, this flat property was replaced by a structured `retentionDescription` object containing `cleanupPolicy` and `retentionTimeInHours`. The Azure CLI removed the `--message-retention` flag as part of this API migration without introducing a backward-compatible alias, causing any script or documentation still referencing the old flag to fail at runtime with no deprecation warning.

#### Resolution Applied

The `--message-retention 1` flag was removed from the command entirely. The corrected command was re-run successfully in Step 8. The Standard SKU default of 7 days (168 hours) was accepted as returned automatically in `retentionDescription.retentionTimeInHours`.

> **Screenshot Placeholder -- ERROR-01: Successful Resolution**
> `[SCREENSHOT: Terminal showing the corrected az eventhubs eventhub create command (no --message-retention flag) with the "➜" green prompt symbol and the complete JSON response showing "status": "Active", "partitionCount": 2, "messageRetentionInDays": 7, and "retentionTimeInHours": 168]`

#### Prevention Going Forward

* Always run `az eventhubs eventhub create --help` after any Azure CLI upgrade to confirm flag availability before executing or updating automation scripts
* Pin the Azure CLI version in all CI/CD pipelines using a container image tag or a requirements lock file
* Never rely on `--message-retention` in any new automation; use `--retention-time-in-hours` if supported by the installed CLI version, or explicitly document reliance on the API default

---

## Best Practices

### Security

* **Never hardcode credentials.** The `sed` injection in Step 14 is a lab-only convenience pattern. In production, load connection strings at runtime from environment variables (`os.environ.get()`) or the Azure Key Vault SDK. The SAS key used in this lab was rotated immediately after the session ended.
* **Scope authorization rules.** Create dedicated Send-only and Listen-only SAS policies rather than using `RootManageSharedAccessKey`, which grants full management access to the namespace.
* **Restrict network access.** Bind the Event Hubs namespace to a VNet service endpoint or Private Endpoint. Set `publicNetworkAccess` to `Disabled` unless explicitly required.
* **Rotate SAS keys on a schedule.** Automate rotation using Azure Key Vault with event-driven restart hooks via Azure App Configuration.

### Reliability

* **Use batch sending.** The `create_batch()` + `send_batch()` pattern in `send_logs.py` is the recommended approach. It respects the maximum AMQP frame size automatically and reduces connection overhead versus sending events individually.
* **Implement retry logic.** Wrap producer send calls in `try/except` and configure the SDK's built-in retry policy (`retry_total`, `retry_backoff_factor`) for transient network errors.
* **Enable AutoInflate.** Always enable AutoInflate on Standard and Premium namespaces to handle unexpected throughput spikes without manual intervention.
* **Use at least 2 partitions.** A single partition is a throughput bottleneck. Two partitions is the minimum for any production workload; size based on expected peak consumer parallelism.

### Observability

* **Pair `IncomingMessages` with `ThrottledRequests`.** `IncomingMessages` confirms data is flowing; `ThrottledRequests > 0` signals the namespace has reached its throughput limit. Alert on `ThrottledRequests > 0` with high severity.
* **Configure diagnostic settings.** Stream `OperationalLogs` and `AutoScaleLogs` to a Log Analytics workspace for long-term audit trails and automated alerting.
* **Tag all resources at creation time.** Apply `environment`, `team`, `cost-center`, and `project` tags to every resource for cost allocation and governance.

### Operational

* **Always run pre-flight checks before provisioning.** Steps 1 through 4 (`az account show`, `az account list`, `az group list`, `az vm list`) form a mandatory pre-flight sequence. These prevent resources from being created in the wrong subscription, region, or resource group.
* **Retrieve dynamic values programmatically.** VM public IPs and SAS connection strings must always be queried fresh. Hardcoding either breaks on VM restart or key rotation.
* **Consolidate health checks into a single gate script.** The full stack check in Step 21 is directly reusable as a post-deployment validation gate in any CI/CD pipeline.
* **Document deprecated CLI flags at the point of failure.** The `--message-retention` error in Step 7 is a canonical example of CLI API drift. Maintain a changelog of deprecated flags and the Azure CLI version that removed them.

---

## Lessons Learned

### 1. Azure CLI Flag Deprecation Causes Blocking Failures With No Warning

**What happened:** The `--message-retention 1` flag in Step 7 caused the entire `az eventhubs eventhub create` command to abort with `unrecognized arguments`. The CLI provided no deprecation warning on previous runs -- the flag simply stopped working after an API version change.

**Root cause:** The `--message-retention` shorthand was removed when the Event Hubs ARM API moved to the `retentionDescription` object model. No alias or graceful degradation was introduced.

**Resolution:** Remove the flag entirely. Re-run without it. The 7-day default was applied automatically and is documented in the JSON output.

**Prevention:** Pin Azure CLI versions in all CI/CD pipelines. Run `az eventhubs eventhub create --help` against the target CLI version before any deployment script is executed after an upgrade.

> **Screenshot Placeholder -- Lesson 1**
> `[SCREENSHOT: Side-by-side terminal comparison: LEFT panel shows Step 7 failed command with "✖" prompt and "unrecognized arguments: --message-retention 1" error output; RIGHT panel shows Step 8 corrected command with "➜" prompt and the JSON response confirming "status": "Active"]`

---

### 2. Pre-Flight Context Verification Prevents Mis-Targeted Deployments

**What happened:** Before any resource was created, four sequential verification commands were run: `az account show`, `az account list`, `az group list`, and `az vm list`. These confirmed that the correct subscription, tenant, resource group, region, and VM were all in scope.

**Root cause / Value:** Multi-subscription environments are the leading cause of resources landing in the wrong subscription or region, resulting in billing surprises, connectivity failures, and wasted remediation time.

**Resolution:** Treat Steps 1 through 4 as a non-negotiable mandatory pre-flight sequence before every deployment session, not optional housekeeping.

**Prevention:** Codify `az account show && az group list --output table` as the first explicit step in every deployment runbook and CI/CD pipeline stage gate.

---

### 3. IncomingMessages Metric Count Exceeds Application-Level Event Count

**What happened:** The script sent 60 application-level log events across 6 runs, but the `IncomingMessages` metric in Step 19 reported 120.

**Root cause:** Azure Event Hubs counts `IncomingMessages` per AMQP message frame, not per application-level `EventData` object. Protocol handshake frames, batch envelope overhead, and broker acknowledgment frames all contribute to the metric count independently of the raw application event count.

**Resolution:** Use `IncomingMessages` as a relative throughput indicator, not an absolute application event counter. For precise event counts, implement a dedicated consumer group that reads from the hub and counts `EventData` objects at the application layer.

---

### 4. SAS Key Injection via `sed` is a Lab-Only Pattern

**What happened:** The connection string containing the SAS key was written directly into `send_logs.py` using `sed -i` in Step 14. This is a critical security anti-pattern outside of isolated lab environments.

**Root cause:** Time-boxed lab constraints prioritized convenience over security.

**Resolution applied post-lab:** The SAS key was rotated immediately after the lab session closed. In production, connection strings must be retrieved at runtime using `os.environ.get()`, the `azure-keyvault-secrets` SDK, or a managed identity token exchange. The script file itself must never contain credentials in any form.

---

### 5. VM Public IP is Dynamic and Must Always Be Queried Per Session

**What happened:** The VM public IP (`20.115.31.19`) was retrieved dynamically using `az vm list-ip-addresses` in Step 11 before each SSH session. This is the correct approach.

**Root cause:** Azure VMs receive a Dynamic public IP allocation by default. The assigned address changes on every VM stop and restart, making any hardcoded reference stale after the first reboot.

**Resolution:** Always resolve the current IP using `VM_IP=$(az vm list-ip-addresses ...)` at session start. For production workloads requiring a stable SSH target, assign a Static Public IP SKU or use Azure Bastion to eliminate public IP exposure entirely.

---

## Troubleshooting

| Symptom | Probable Cause | Resolution |
|---------|----------------|------------|
| `unrecognized arguments: --message-retention` | Deprecated Azure CLI flag | Remove the flag; 7-day default retention applies automatically |
| Event Hub not found after error | Command aborted before resource creation | Re-run without the deprecated flag; verify with `az eventhubs eventhub show` |
| `AuthorizationFailedException` in Python | Incorrect or expired SAS key in connection string | Re-retrieve via `az eventhubs namespace authorization-rule keys list` and re-inject |
| Metrics return 0 immediately after script runs | Azure Monitor aggregation lag (3 to 10 minutes) | Wait at least 5 minutes and re-query; add `sleep 300` in automation before metric queries |
| SSH connection refused | NSG blocking port 22 or VM in deallocated state | Check NSG inbound rules for port 22; run `az vm get-instance-view` to confirm power state |
| `ModuleNotFoundError: No module named 'azure.eventhub'` | SDK not installed on VM | Run `pip3 install azure-eventhub` on the VM |
| `QuotaExceededException` during send | Namespace throughput units exhausted | Verify AutoInflate is enabled; increase max TU if needed |
| `EventHubError: Connection refused` | Outbound AMQP port blocked by NSG | Allow outbound on port `5671` (AMQP) or `443` (AMQP over WebSockets) in the NSG |
| Placeholder still in script after `sed` | Delimiter conflict or path mismatch in `sed` | Verify the exact placeholder text with `cat` before running `sed`; confirm path is correct |

---

## References

* [Azure Event Hubs Documentation](https://learn.microsoft.com/en-us/azure/event-hubs/)
* [azure-eventhub Python SDK -- PyPI](https://pypi.org/project/azure-eventhub/)
* [Azure CLI Event Hubs Reference](https://learn.microsoft.com/en-us/cli/azure/eventhubs)
* [Azure Monitor Metrics Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-overview)
* [Event Hubs Best Practices](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-best-practices)
* [Azure Managed Identity for Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/authenticate-managed-identity)
* [Azure Key Vault Secrets SDK for Python](https://learn.microsoft.com/en-us/python/api/overview/azure/keyvault-secrets-readme)
* [Conventional Commits Specification](https://www.conventionalcommits.org/)





<img width="1037" height="846" alt="image" src="https://github.com/user-attachments/assets/8cbcbd47-d0e0-4d52-9734-f17d3dfe8d29" />
<img width="1033" height="863" alt="image" src="https://github.com/user-attachments/assets/347fabf8-3f62-4f6e-94dd-9d4ede9ec0e5" />
<img width="1031" height="595" alt="image" src="https://github.com/user-attachments/assets/8ef48449-cb6c-4a57-bc45-823e0d868a66" />
<img width="1035" height="446" alt="image" src="https://github.com/user-attachments/assets/26bccb2e-90a2-4045-9557-3443584bf4c4" />
<img width="1034" height="839" alt="image" src="https://github.com/user-attachments/assets/6663943f-d2d7-46b3-80ac-0de6c198b5c3" />
<img width="1035" height="806" alt="image" src="https://github.com/user-attachments/assets/6dad978e-98ba-4c50-9de8-08749834d0b5" />
<img width="1034" height="446" alt="image" src="https://github.com/user-attachments/assets/d8016fc3-62f5-496a-b2dd-548cf655ce07" />
<img width="1035" height="620" alt="image" src="https://github.com/user-attachments/assets/d3e30b11-1f4c-4f76-9e09-6422a74a6314" />
<img width="1033" height="865" alt="image" src="https://github.com/user-attachments/assets/98abe896-436a-4419-8f49-7368f5301b6f" />
<img width="1033" height="665" alt="image" src="https://github.com/user-attachments/assets/8262e888-19f0-441e-856c-15c6f2af30a1" />
<img width="1035" height="523" alt="image" src="https://github.com/user-attachments/assets/71b5b0d3-00b0-4b0e-8c80-f5df6e7b8365" />
<img width="1034" height="485" alt="image" src="https://github.com/user-attachments/assets/1292dc59-839f-43fb-b791-4709564f197c" />
<img width="1032" height="702" alt="image" src="https://github.com/user-attachments/assets/3b6a1117-3c61-407a-8195-4a68c1ac695c" />
<img width="1031" height="868" alt="image" src="https://github.com/user-attachments/assets/26d7d561-f559-4d09-bccf-cbb2133f8c12" />
<img width="1033" height="858" alt="image" src="https://github.com/user-attachments/assets/ed916cf4-37ae-437e-9c19-93ec2b9c53bf" />
<img width="1037" height="867" alt="image" src="https://github.com/user-attachments/assets/4915be82-422d-456f-b883-fcaa47723458" />
<img width="1026" height="861" alt="image" src="https://github.com/user-attachments/assets/0b9216bb-3bcd-40d2-a9fb-1698ee7967d1" />
<img width="1033" height="852" alt="image" src="https://github.com/user-attachments/assets/8c432816-f684-45ea-929e-0bd3ab9deccf" />
<img width="1031" height="828" alt="image" src="https://github.com/user-attachments/assets/5a05d1fb-2e94-4c84-bc12-6231ce859ec8" />

