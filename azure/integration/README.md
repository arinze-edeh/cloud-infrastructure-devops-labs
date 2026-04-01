# Azure Integration

[![Azure](https://img.shields.io/badge/Azure-Integration-0078D4?style=flat-square&logo=microsoftazure)](https://azure.microsoft.com)
[![Python](https://img.shields.io/badge/Python-3.10-3776AB?style=flat-square&logo=python)](https://www.python.org/)
[![PHP](https://img.shields.io/badge/PHP-8.x-777BB4?style=flat-square&logo=php)](https://www.php.net)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04_LTS-E95420?style=flat-square&logo=ubuntu)](https://ubuntu.com)
[![Status](https://img.shields.io/badge/Status-Production_Ready-brightgreen?style=flat-square)]()

---

## Overview

This directory covers real-world Azure integration patterns spanning event streaming, cross-service connectivity, and centralized log collection. Each project addresses a concrete operational gap: siloed VM logs, isolated application tiers, and the absence of durable observability pipelines.

Work here reflects production-relevant constraints including policy-restricted environments, multi-region VM deployments, SDK version compatibility, and Azure CLI API changes. All tasks were executed hands-on via Azure CLI, SSH, and Python SDKs with full verification against Azure Monitor metrics and live service endpoints.

---

## Directory Structure
```
azure/integration/
├── centralized-telemetry-ingestion-pipeline/     # VM-to-Event Hubs log streaming (Python SDK)
├── cross-vm-application-database-integration/    # PHP app to MySQL cross-region connectivity
├── event-driven-log-aggregation-system/          # VM-to-Event Hubs with Blob Storage backup
└── README.md
```

---

## Project Summaries

---

### [Centralized Telemetry Ingestion Pipeline](./centralized-telemetry-ingestion-pipeline/)

**Quick Summary:** Streams structured application logs from an Ubuntu VM to Azure Event Hubs using a Python producer client. Validated end-to-end via Azure Monitor metrics confirming 120 incoming message frames and 8,569 bytes ingested across 6 script runs.

**Purpose:** Eliminate log silos on individual VMs by routing workload telemetry into a scalable, durable event streaming service ready for downstream processing and alerting.

**Approach:**
- Provisioned `datacenter-namespace` (Standard SKU, AutoInflate enabled, max 10 TU) and `datacenter-hub` (2 partitions, 7-day retention) via Azure CLI
- Retrieved the SAS connection string and injected it into the pre-existing `send_logs.py` using `sed -i` for lab-scoped credential binding
- Executed the `EventHubProducerClient` script 6 times (3 individual runs, 3 via loop) sending 10 events per run
- Confirmed delivery via `az monitor metrics list` querying both `IncomingMessages` and `IncomingBytes`

**Key Decisions:**
- `--message-retention` flag removed after encountering a blocking CLI deprecation error; 7-day default was accepted from the API and documented in the error log
- `IncomingMessages` (120) vs. application events (60) discrepancy traced to AMQP framing overhead, not data loss
- `create_batch()` + `send_batch()` pattern used throughout for AMQP frame-size compliance and reduced connection overhead

**Outcome:** Fully operational log pipeline confirmed via dual Azure Monitor metrics. Full stack health verified across namespace, Event Hub, and VM in a single gate command.

---

### [Cross-VM Application Database Integration](./cross-vm-application-database-integration/)

**Quick Summary:** Connects a PHP application on an East US VM to a MySQL database on a separate Central US VM over Azure public networking. Validated by a browser returning `Connected successfully` from a live PHP endpoint.

**Purpose:** Validate cross-region, cross-VM database connectivity for the Nautilus DevOps team, establishing a working integration pattern for geographically distributed Azure workloads.

**Approach:**
- Deployed `devops-mysql-vm` (MySQL 5.7 on Ubuntu, Jetware image) in Central US via Azure Marketplace with port 3306 added post-deployment to the NSG at priority 310
- Created `devops_db`, `devops_user` with `'%'` host wildcard, and applied `FLUSH PRIVILEGES`
- Confirmed MySQL listening on `:::3306` via `ss -tlnp`
- Updated `/var/www/html/db_test.php` on the PHP VM with correct host IP, credentials, and database name via `nano`
- Resolved SSH access issue on the PHP VM caused by key-only authentication provisioning; password reset via Azure Portal

**Key Decisions:**
- `'devops_user'@'%'` wildcard used to permit connections from any source IP, avoiding the silent failure caused by `@'localhost'` in remote connection scenarios
- NSG rule added at the Azure layer only after confirming `ufw` was inactive on the MySQL VM
- VM size `Standard_D2s_v3` placed in Zone 2 after Zone 1 availability failure during provisioning

**Outcome:** PHP endpoint returned `Connected successfully` in-browser and via `curl`, confirming live TCP connectivity on port 3306 across two Azure regions.

---

### [Event-Driven Log Aggregation System](./event-driven-log-aggregation-system/)

**Quick Summary:** Provisions a full log pipeline from scratch: VM creation, Event Hubs namespace, and Blob Storage with a Python script publishing logs simultaneously to both targets. Confirms delivery via blob listing and Azure Monitor metrics.

**Purpose:** Provide the Nautilus DevOps team with durable log archival and real-time event streaming in a single pipeline, covering both compliance retention (Blob) and live observability (Event Hubs).

**Approach:**
- Dynamically resolved the resource group via `az group list` to avoid hardcoded scope assumptions
- Provisioned `xfusion-namespace`, `xfusion-hub`, storage account `xfusionst6179` (Standard_LRS), and blob container `xfusion-backup-23374` in sequence
- Enabled `allowBlobPublicAccess` on the storage account after a silent `"created": false` on first container creation attempt
- Transferred `send_logs.py` to the VM via `scp`, installed `azure-eventhub` and `azure-storage-blob` via `pip3`, and injected both connection strings via `sed -i` over SSH
- Verified delivery via `az storage blob list` (confirming `AppendBlob` type and 18-byte payload) and `az monitor metrics list` (confirming `IncomingMessages: 1.0`)

**Key Decisions:**
- `AppendBlob` type confirmed in output, validating that the script used the correct blob client for sequential log writes without overwrite risk
- `|` delimiter selected for `sed` to avoid conflicts with forward slashes in connection string URLs
- `StrictHostKeyChecking=no` used in `scp` and SSH commands for non-interactive pipeline execution, with the host fingerprint permanently added to `known_hosts`

**Outcome:** Single script execution confirmed logs delivered to both Event Hubs and Blob Storage. Azure Monitor metric delay (~2 minutes) documented as a known operational consideration.

---

## Technologies and Tools

| Layer | Tools |
|---|---|
| Cloud Platform | Microsoft Azure (Event Hubs, Blob Storage, VMs, NSG, Azure Monitor) |
| CLI | Azure CLI (`az eventhubs`, `az storage`, `az vm`, `az monitor`) |
| Runtime | Python 3.10, PHP 8.x, Ubuntu 22.04 LTS / 16.04 LTS |
| SDKs | `azure-eventhub 5.15.1`, `azure-storage-blob 12.28.0`, `azure-core 1.39.0` |
| Database | MySQL 5.7 (Jetware Marketplace image) |
| Web Server | Apache2 |
| Scripting | Bash, `sed`, `scp`, `ssh`, `for` loops |
| Observability | Azure Monitor Metrics (`IncomingMessages`, `IncomingBytes`) |

---

## Key Outcomes and Skills Demonstrated

- **Azure CLI fluency:** Multi-step provisioning across Event Hubs, Blob Storage, NSG, and VM resources with dynamic resource group resolution and focused `--query` projections
- **Python SDK integration:** `EventHubProducerClient`, `AppendBlobClient`, and `from_connection_string()` patterns applied in production-aligned configurations
- **Error diagnosis and recovery:** Documented and resolved a blocking Azure CLI API deprecation (`--message-retention`), silent storage container creation failure, and SSH authentication mismatch; each with root cause analysis and prevention guidance
- **Cross-service pipeline design:** End-to-end log flow from VM workload to Event Hubs and Blob Storage with dual-channel verification
- **Cross-region connectivity:** MySQL-to-PHP integration across East US and Central US with NSG rule management and MySQL host wildcard configuration
- **Observability validation:** Azure Monitor metric queries used as a deployment gate, with documented understanding of AMQP frame counting behavior and metric ingestion lag
- **Security awareness:** SAS key rotation, `RootManageSharedAccessKey` scoping, `sed`-based credential injection documented as lab-only with Managed Identity alternatives provided

---

## How to Navigate

Each subdirectory contains a standalone `README.md` with:

- Full CLI command sequences with actual terminal output
- Inline screenshot placeholders at every verification step
- A dedicated errors section with root cause, resolution, and prevention guidance
- Configuration reference tables for all key resource parameters
- Best practices and lessons learned sections written for production applicability

Start with the project most relevant to your use case:

- **Event streaming only:** [Centralized Telemetry Ingestion Pipeline](./centralized-telemetry-ingestion-pipeline/)
- **Cross-VM application connectivity:** [Cross-VM Application Database Integration](./cross-vm-application-database-integration/)
- **Dual-target log pipeline (streaming + archival):** [Event-Driven Log Aggregation System](./event-driven-log-aggregation-system/)

---

> All labs executed in Azure Free Labs (`Azure Free Labs` subscription, East US / Central US regions). SAS keys and connection strings shown in individual READMEs are rotated and expired. Do not use them in any environment.
