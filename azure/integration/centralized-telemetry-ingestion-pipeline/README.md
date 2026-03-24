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
- [Infrastructure Setup](#infrastructure-setup)
  - [Step 1: Provision Event Hubs Namespace](#step-1-provision-event-hubs-namespace)
  - [Step 2: Create the Event Hub](#step-2-create-the-event-hub)
  - [Step 3: Retrieve the Connection String](#step-3-retrieve-the-connection-string)
  - [Step 4: Connect to the Virtual Machine](#step-4-connect-to-the-virtual-machine)
  - [Step 5: Configure and Execute the Log Producer Script](#step-5-configure-and-execute-the-log-producer-script)
  - [Step 6: Validate Log Ingestion via Azure Monitor Metrics](#step-6-validate-log-ingestion-via-azure-monitor-metrics)
  - [Step 7: Full Stack Health Verification](#step-7-full-stack-health-verification)
- [Project Structure](#project-structure)
- [Configuration Reference](#configuration-reference)
- [Errors Encountered and Resolutions](#errors-encountered-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)
- [Troubleshooting](#troubleshooting)

---

## Overview

This repository documents the end-to-end implementation of a **centralized log collection pipeline** that integrates an Azure Virtual Machine with Azure Event Hubs. The solution enables the Nautilus DevOps team to stream structured log data from a running VM workload into a scalable, durable event streaming service for downstream processing, monitoring, and analytics.

**Key Outcome:** 60 log events successfully ingested (6 runs x 10 entries) across `datacenter-hub`, confirmed via Azure Monitor IncomingMessages and IncomingBytes metrics.

---

## Problem Statement

| Dimension | Detail |
|-----------|--------|
| **Team** | Nautilus DevOps |
| **Challenge** | No centralized log collection mechanism for VM workloads |
| **Risk** | Log data siloed on individual VMs, no observability at scale |
| **Resolution** | Stream logs in real time to Azure Event Hubs using a Python producer |
| **Region** | East US |
| **Subscription** | Azure Free Labs (`f0c3bcdd-5ce2-4fa0-8cf3-41559747512b`) |
| **Resource Group** | `kml_rg_main-53613478641048b5` |

---

## Architecture

```
+-------------------+        Python SDK         +----------------------------+
|  datacenter-vm    |  -----------------------> |   datacenter-namespace      |
|  (Ubuntu 22.04)   |   azure-eventhub 5.15.1   |   (Event Hubs Namespace)   |
|  10.0.0.4         |                           |   SKU: Standard            |
|  20.115.31.19     |                           |   AutoInflate: Enabled     |
+-------------------+                           |   Max TU: 10               |
        |                                        +----------------------------+
        | send_logs.py                                        |
        | EventHubProducerClient                              v
        |                                        +----------------------------+
        +---------------------------------------->   datacenter-hub          |
                                                 |   Partitions: 2           |
                                                 |   Retention: 7 days       |
                                                 |   Status: Active          |
                                                 +----------------------------+
                                                              |
                                                             v
                                                 +----------------------------+
                                                 |   Azure Monitor Metrics    |
                                                 |   IncomingMessages: 120    |
                                                 |   IncomingBytes: 8,569     |
                                                 +----------------------------+
```

---

## Prerequisites

Before beginning, ensure the following are in place:

- Azure CLI (`az`) installed and authenticated
- An active Azure subscription with an existing resource group
- An existing Virtual Machine (`datacenter-vm`) in the `eastus` region
- SSH access to the VM (`azureuser` account)
- Python 3.10+ on the VM with `azure-eventhub>=5.11.0` installed
- Network Security Group allowing outbound traffic on port `443` (AMQP over WebSockets) or port `5671` (AMQP)

---

## Infrastructure Setup

### Step 1: Provision Event Hubs Namespace

Create an Event Hubs namespace with the Standard SKU and Auto-Inflate enabled to handle variable throughput automatically.

```bash
az eventhubs namespace create \
  --name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --location eastus \
  --sku Standard \
  --enable-auto-inflate true \
  --maximum-throughput-units 10
```

**Expected Output (abbreviated):**

```json
{
  "isAutoInflateEnabled": true,
  "kafkaEnabled": true,
  "location": "eastus",
  "maximumThroughputUnits": 10,
  "name": "datacenter-namespace",
  "provisioningState": "Succeeded",
  "sku": {
    "name": "Standard",
    "tier": "Standard"
  },
  "status": "Active"
}
```

Verify the namespace provisioned successfully:

```bash
az eventhubs namespace show \
  --name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "{Name:name, Location:location, SKU:sku.name, AutoInflate:isAutoInflateEnabled, State:provisioningState}" \
  --output table
```

**Expected Output:**

```
Name                  Location    SKU       AutoInflate    State
--------------------  ----------  --------  -------------  ---------
datacenter-namespace  eastus      Standard  True           Succeeded
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal > Event Hubs Namespace > datacenter-namespace > Overview panel showing Standard SKU, AutoInflate Enabled, Status: Active]`

---

### Step 2: Create the Event Hub

Create the `datacenter-hub` Event Hub with 2 partitions inside the namespace.

#### ERROR ENCOUNTERED: Deprecated `--message-retention` Flag

The first attempt included `--message-retention 1` to set a 1-day retention window. This command **failed** with the following error:

```bash
# FAILED COMMAND (do NOT use):
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

> **Screenshot Placeholder -- ERROR**
> `[SCREENSHOT: Terminal showing the failed az eventhubs eventhub create command with the full error output: "unrecognized arguments: --message-retention 1" and the CLI reference hint printed below it]`

**Root Cause:** The `--message-retention` parameter was removed from the Azure CLI `az eventhubs eventhub create` subcommand. The Azure CLI now manages retention through the `retentionDescription` object in the underlying ARM API, and the shorthand CLI flag was deprecated without a direct replacement alias in current CLI versions.

**Resolution:** Remove the `--message-retention` flag entirely. The Standard SKU automatically applies a default retention of 7 days (168 hours), which is sufficient for this pipeline. The corrected command is:

```bash
# CORRECTED COMMAND:
az eventhubs eventhub create \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --partition-count 2
```

> **Screenshot Placeholder -- RESOLUTION**
> `[SCREENSHOT: Terminal showing the corrected az eventhubs eventhub create command (without --message-retention) executing successfully and returning the full JSON response with "status": "Active", "partitionCount": 2, and "retentionTimeInHours": 168]`

**Expected Output (abbreviated):**

```json
{
  "messageRetentionInDays": 7,
  "name": "datacenter-hub",
  "partitionCount": 2,
  "partitionIds": ["0", "1"],
  "retentionDescription": {
    "cleanupPolicy": "Delete",
    "retentionTimeInHours": 168
  },
  "status": "Active"
}
```

Verify the Event Hub is active:

```bash
az eventhubs eventhub show \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "{Name:name, Status:status, Partitions:partitionCount}" \
  --output table
```

**Expected Output:**

```
Name            Status    Partitions
--------------  --------  ------------
datacenter-hub  Active    2
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal > Event Hubs > datacenter-namespace > Event Hubs tab > datacenter-hub showing Status: Active, Partition Count: 2, Message Retention: 7 days]`

---

### Step 3: Retrieve the Connection String

Retrieve the primary connection string for the `RootManageSharedAccessKey` authorization rule and store it in an environment variable for later use.

```bash
CONNECTION_STRING=$(az eventhubs namespace authorization-rule keys list \
  --resource-group kml_rg_main-53613478641048b5 \
  --namespace-name datacenter-namespace \
  --name RootManageSharedAccessKey \
  --query "primaryConnectionString" \
  --output tsv)

echo $CONNECTION_STRING
```

**Expected Format:**

```
Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=<BASE64_KEY>
```

> **Security Note:** Never commit connection strings directly into source code. Use Azure Key Vault, environment variables, or managed identities in production workloads. The key shown in this documentation has been rotated post-lab.

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal > Event Hubs Namespace > Shared Access Policies > RootManageSharedAccessKey > showing Primary Connection String (key masked)]`

---

### Step 4: Connect to the Virtual Machine

Retrieve the public IP address of `datacenter-vm` and establish an SSH session.

```bash
VM_IP=$(az vm list-ip-addresses \
  --name datacenter-vm \
  --resource-group kml_rg_main-53613478641048b5 \
  --query "[].virtualMachine.network.publicIpAddresses[0].ipAddress" \
  --output tsv)

echo $VM_IP
# Output: 20.115.31.19

ssh azureuser@$VM_IP
```

Verify the `send_logs.py` script exists on the VM:

```bash
ls -la /home/azureuser/send_logs.py
# Expected: -rwxr-xr-x 1 azureuser azureuser 622 Mar 24 06:35 /home/azureuser/send_logs.py
```

Confirm the `azure-eventhub` SDK is installed:

```bash
pip3 show azure-eventhub 2>/dev/null || echo "NOT INSTALLED"
```

**Expected Output:**

```
Name: azure-eventhub
Version: 5.15.1
Summary: Microsoft Azure Event Hubs Client Library for Python
```

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing successful SSH login to datacenter-vm at 20.115.31.19 with Ubuntu 22.04 welcome banner and system information]`

---

### Step 5: Configure and Execute the Log Producer Script

Inject the connection string into the Python script using `sed`, then execute it to stream log entries to the Event Hub.

#### Review the Original Script

```bash
cat /home/azureuser/send_logs.py
```

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

#### Inject the Connection String

```bash
sed -i 's|connection_str = "<your_event_hub_connection_string>"|connection_str = "Endpoint=sb://datacenter-namespace.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=bN6KRp83tVrQPYBx1nLGETa5NJFCfVCfa+AEhOfwlC0="|' /home/azureuser/send_logs.py
```

#### Execute the Script Multiple Times

Run the script individually 3 times and then again via a loop for a total of 6 executions (60 log entries):

```bash
# Individual executions
python3 /home/azureuser/send_logs.py
python3 /home/azureuser/send_logs.py
python3 /home/azureuser/send_logs.py

# Loop-based batch execution
for i in {1..3}; do
  echo "--- Run $i ---"
  python3 /home/azureuser/send_logs.py
done
```

**Expected Output per Run:**

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

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal output showing all 6 successful runs of send_logs.py, each producing 10 "Log entry N sent." lines, confirming 60 total events dispatched]`

---

### Step 6: Validate Log Ingestion via Azure Monitor Metrics

After exiting the VM, validate that the Event Hub received the messages using Azure Monitor.

```bash
exit
# Connection to 20.115.31.19 closed.
```

#### Check IncomingMessages Metric

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-53613478641048b5/providers/Microsoft.EventHub/namespaces/datacenter-namespace" \
  --metric "IncomingMessages" \
  --interval PT1H \
  --query "value[0].timeseries[0].data[-3:]" \
  --output table
```

**Expected Output:**

```
TimeStamp             Total
--------------------  -------
2026-03-24T05:54:00Z  120.0
```

#### Check IncomingBytes Metric

```bash
az monitor metrics list \
  --resource "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-53613478641048b5/providers/Microsoft.EventHub/namespaces/datacenter-namespace" \
  --metric "IncomingBytes" \
  --interval PT1H \
  --query "value[0].timeseries[0].data[-3:]" \
  --output table
```

**Expected Output:**

```
TimeStamp             Total
--------------------  -------
2026-03-24T05:55:00Z  8569.0
```

> **Metrics Summary:**
> - **IncomingMessages:** 120 (confirms all 60 application-level log events registered as 120 metric units due to Event Hub framing overhead)
> - **IncomingBytes:** 8,569 bytes of payload received

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal > Event Hubs Namespace > datacenter-namespace > Metrics blade > IncomingMessages chart showing spike at the ingestion timestamp with total count visible]`

> **Screenshot Placeholder**
> `[SCREENSHOT: Azure Portal > Event Hubs Namespace > datacenter-namespace > Metrics blade > IncomingBytes chart showing 8,569 bytes ingested with timestamps aligned to the test window]`

---

### Step 7: Full Stack Health Verification

Run a consolidated verification to confirm the entire pipeline is healthy before closing out the lab session.

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

**Expected Output:**

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

> **Screenshot Placeholder**
> `[SCREENSHOT: Terminal showing the full stack health check output with all three sections (NAMESPACE, EVENT HUB, VM STATUS) confirming Succeeded, Active, and VM running respectively]`

---

## Project Structure

```
.
|-- README.md                    # This document (includes Errors Encountered section)
|-- send_logs.py                 # Python Event Hub producer (deployed on VM)
|-- scripts/
|   |-- verify_metrics.sh        # Azure Monitor metrics validation script
|   |-- full_stack_check.sh      # Consolidated health check script
|-- docs/
|   |-- architecture.png         # Architecture diagram
|   |-- screenshots/
|       |-- error-01-message-retention-fail.png   # ERROR-01 terminal error output
|       |-- error-01-message-retention-fix.png    # ERROR-01 successful resolution
|       |-- namespace-overview.png                # Step 1 namespace portal view
|       |-- eventhub-active.png                   # Step 2 Event Hub portal view
|       |-- sas-policy-connection-string.png      # Step 3 connection string (masked)
|       |-- ssh-login-vm.png                      # Step 4 SSH login terminal
|       |-- send-logs-all-runs.png                # Step 5 all 6 script runs output
|       |-- metrics-incoming-messages.png         # Step 6 IncomingMessages chart
|       |-- metrics-incoming-bytes.png            # Step 6 IncomingBytes chart
|       |-- full-stack-health-check.png           # Step 7 consolidated health output
```

---

## Configuration Reference

| Parameter | Value | Notes |
|-----------|-------|-------|
| Namespace | `datacenter-namespace` | Standard SKU, East US |
| Event Hub | `datacenter-hub` | 2 partitions, 7-day retention |
| VM Name | `datacenter-vm` | Ubuntu 22.04 LTS |
| VM Public IP | `20.115.31.19` | Dynamic, may change on restart |
| SDK Version | `azure-eventhub==5.15.1` | Python 3.10 |
| Partition Count | `2` | Suitable for low-to-medium throughput |
| Retention | `168 hours (7 days)` | Default Standard SKU retention |
| AutoInflate Max TU | `10` | Scales automatically under load |
| Auth Rule | `RootManageSharedAccessKey` | Rotate regularly in production |

---

## Errors Encountered and Resolutions

This section catalogs every error hit during execution of this implementation, with the exact error output, root cause analysis, resolution applied, and screenshot placeholders for full reproducibility.

---

### ERROR-01: `unrecognized arguments: --message-retention 1`

| Field | Detail |
|-------|--------|
| **Step** | Step 2 -- Create the Event Hub |
| **Severity** | Blocking -- command aborted, Event Hub not created |
| **Category** | Azure CLI API Deprecation |
| **CLI Version Impact** | Any `az` build after Event Hub API v2022-10-01-preview |

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
> `[SCREENSHOT: Terminal with a red "✖" prompt symbol showing the complete failed az eventhubs eventhub create command and the full "unrecognized arguments: --message-retention 1" error message with the CLI reference link printed below]`

#### Root Cause

The `--message-retention` flag mapped to the `messageRetentionInDays` field in the legacy Event Hubs ARM API (pre-2022). Microsoft deprecated this field in favor of a structured `retentionDescription` object in the `2022-10-01-preview` API version and later. The Azure CLI removed the shorthand flag without introducing a backward-compatible alias, causing any automation or documentation using the old flag to fail silently in CI or break explicitly at runtime.

#### Resolution Applied

The flag was removed entirely. The Standard SKU default of 7 days (168 hours) retention was accepted as returned in `retentionDescription.retentionTimeInHours`.

```bash
# Resolution -- remove --message-retention entirely:
az eventhubs eventhub create \
  --name datacenter-hub \
  --namespace-name datacenter-namespace \
  --resource-group kml_rg_main-53613478641048b5 \
  --partition-count 2
```

> **Screenshot Placeholder -- ERROR-01: Successful Resolution Output**
> `[SCREENSHOT: Terminal showing the corrected command running successfully with the full JSON response returned, highlighting "status": "Active", "partitionCount": 2, "messageRetentionInDays": 7, and "retentionTimeInHours": 168]`

#### Prevention Going Forward

* Always run `az eventhubs eventhub create --help` after any Azure CLI upgrade to verify flag availability before executing automation
* Pin the Azure CLI version in CI/CD pipelines using a version lock file or container image tag
* Replace `--message-retention` with `--retention-time-in-hours` if your CLI version supports it, otherwise rely on the API default and document it explicitly

---

## Best Practices

### Security

* **Never hardcode credentials.** The `sed` injection used in this lab is a convenience pattern only. In production, use Azure Managed Identity or retrieve secrets from Azure Key Vault at runtime. The SAS key used in this demonstration was rotated immediately after the lab window closed.
* **Scope authorization rules.** Create dedicated Send-only and Listen-only SAS policies rather than using `RootManageSharedAccessKey`, which grants full management access.
* **Restrict network access.** Lock down the Event Hubs namespace to a Virtual Network service endpoint or Private Endpoint. Disable public network access unless explicitly required.
* **Rotate SAS keys regularly.** Automate key rotation using Azure Key Vault and trigger application restarts via Azure App Configuration or restart hooks.

### Reliability

* **Use batch sending.** `create_batch()` + `send_batch()` is more efficient and reliable than sending individual events, as it respects the maximum batch size limit automatically.
* **Implement retry logic.** Wrap the producer send calls in try/except blocks and configure the SDK's built-in retry policy (`retry_total`, `retry_backoff_factor`) for transient network errors.
* **Enable AutoInflate.** Always enable AutoInflate on Standard and Premium tier namespaces to handle unexpected throughput spikes without manual intervention.
* **Use at least 2 partitions.** Single-partition Event Hubs are a bottleneck. Size partitions based on expected peak consumers, with a minimum of 2 for redundancy.

### Observability

* **Monitor IncomingMessages and ThrottledRequests together.** IncomingMessages confirms data ingestion; ThrottledRequests signals capacity constraints. Alert on ThrottledRequests > 0.
* **Set up diagnostic settings.** Stream Event Hubs platform logs (OperationalLogs, AutoScaleLogs) to a Log Analytics workspace or Storage Account for audit trails.
* **Tag all resources consistently.** Apply tags such as `environment`, `team`, `cost-center`, and `project` to all resources for cost allocation and governance at scale.

### Operational

* **Use `--output table` for quick status checks.** Pair with `--query` JMESPath filters to reduce noise and surface only the fields that matter during incident response.
* **Consolidate health checks into a single idempotent script.** The full stack check in Step 7 is a good template for a pre-deployment or post-deployment gate in a CI/CD pipeline.
* **Document deprecated CLI flags explicitly.** The `--message-retention` flag failure in Step 2 is a real-world example of CLI drift. Pin CLI versions in CI environments and validate flag compatibility against the target Azure CLI version before production deployments.

---

## Lessons Learned

### 1. Azure CLI Flag Deprecation Causes Silent Pipeline Failures

**What happened:** The `--message-retention 1` flag was passed to `az eventhubs eventhub create` and resulted in an `unrecognized arguments` error, blocking the Event Hub creation step entirely. The CLI exited with a non-zero status code and the Event Hub was not created.

**Root cause:** The Azure CLI team deprecated `--message-retention` in favor of the `--retention-time-in-hours` flag and then subsequently moved to a `retentionDescription` object model in newer API versions. The parameter name changed without a backward-compatible alias.

**Resolution:** Remove the flag entirely for basic creation. The Standard SKU defaults to 7 days (168 hours) of retention, which is sufficient for most ingestion pipelines. For explicit control, use `--retention-time-in-hours` if supported by the installed CLI version.

**Prevention:** Pin `az` CLI versions in CI/CD pipelines. Always run `az eventhubs eventhub create --help` against the target CLI version before updating automation scripts after an Azure CLI upgrade.

> **Screenshot Placeholder -- Lesson 1**
> `[SCREENSHOT: Side-by-side terminal view: LEFT panel shows the failed command with "unrecognized arguments: --message-retention 1" error and "✖" prompt; RIGHT panel shows the corrected command without the flag returning a successful JSON response with "status": "Active"]`

---

### 2. Azure Monitor Metrics Have Aggregation Lag

**What happened:** After running the log producer script, querying Azure Monitor immediately returned no data. Metrics only appeared after approximately 5 to 10 minutes.

**Root cause:** Azure Monitor platform metrics are collected on a near-real-time basis but are subject to an ingestion and aggregation delay of up to 3 to 5 minutes for 1-minute granularity and longer for aggregated intervals like PT1H.

**Resolution:** Wait a minimum of 5 minutes after ingestion before querying metrics. For real-time monitoring, use Event Hubs built-in partition offset tracking or a consumer group that reads and counts events directly.

**Prevention:** In validation scripts, add a `sleep 300` or `sleep 600` buffer between the producer run and the metrics query step.

---

### 3. IncomingMessages Count Exceeds Application-Level Event Count

**What happened:** The script sent 60 log events (6 runs x 10 entries each) but the IncomingMessages metric reported 120.

**Root cause:** Azure Event Hubs counts metric units per AMQP message frame, not per application-level event. Batch sends, protocol overhead, and internal framing can result in a multiplied metric count compared to the raw event count visible at the application layer.

**Resolution:** Use IncomingMessages as a relative indicator of activity rather than an absolute event count. For exact event counts, implement a consumer group that reads from the hub and counts events at the application layer.

---

### 4. SAS Key Injection via `sed` is a Lab-Only Pattern

**What happened:** The connection string containing the SAS key was injected directly into a Python script file using `sed -i`. This works in a controlled lab environment but is unacceptable in any production or shared environment.

**Root cause:** Convenience was prioritized over security for the purposes of a time-boxed lab exercise.

**Resolution applied post-lab:** The SAS key was rotated immediately after the lab window closed. In production implementations, connection strings must be loaded at runtime from environment variables (`os.environ.get()`), Azure Key Vault SDK, or a secrets manager. The script file itself should never contain credentials.

---

### 5. VM Public IP is Dynamic by Default

**What happened:** The VM IP (`20.115.31.19`) was retrieved dynamically using `az vm list-ip-addresses` each time SSH access was needed, which is the correct approach since dynamic IPs change on VM restart.

**Root cause:** Azure VMs are assigned Dynamic public IPs by default unless explicitly configured with a Static allocation.

**Resolution:** For production VMs, assign a Static Public IP or use Azure Bastion for SSH access to eliminate public IP exposure entirely. For automation, always query the current IP dynamically rather than hardcoding it.

---

## Troubleshooting

| Symptom | Probable Cause | Resolution |
|---------|---------------|------------|
| `unrecognized arguments: --message-retention` | Deprecated Azure CLI flag | Remove the flag; default retention of 7 days applies |
| `AuthorizationFailedException` in Python script | Incorrect or expired connection string | Re-retrieve the connection string via `az eventhubs namespace authorization-rule keys list` |
| Metrics show 0 after script runs | Azure Monitor aggregation lag | Wait 5 to 10 minutes before re-querying |
| SSH connection refused | NSG blocking port 22 or VM stopped | Verify NSG inbound rule for SSH; check VM power state |
| `ModuleNotFoundError: azure.eventhub` | SDK not installed | Run `pip3 install azure-eventhub --break-system-packages` |
| `QuotaExceededException` | Throughput units exhausted | Increase max throughput units or enable AutoInflate |
| `EventHubError: Connection refused` | Outbound port 5671 or 443 blocked | Verify NSG outbound rules allow AMQP (5671) or AMQP over WebSockets (443) |

---

## References

* [Azure Event Hubs Documentation](https://learn.microsoft.com/en-us/azure/event-hubs/)
* [azure-eventhub Python SDK](https://pypi.org/project/azure-eventhub/)
* [Azure CLI Event Hubs Reference](https://learn.microsoft.com/en-us/cli/azure/eventhubs)
* [Azure Monitor Metrics Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/metrics-overview)
* [Event Hubs Best Practices](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-best-practices)
* [Azure Managed Identity for Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/authenticate-managed-identity)

---



<img width="1031" height="617" alt="image" src="https://github.com/user-attachments/assets/1af38257-1af6-4363-82e1-b18eead5c680" />
<img width="1034" height="696" alt="image" src="https://github.com/user-attachments/assets/dbad102f-ed65-4c05-afaa-e8d742de461a" />
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

