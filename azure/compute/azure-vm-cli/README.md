# [Provisioning Azure Virtual Machines via CLI: Infrastructure-as-Command Automation](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> Deploying a production-ready Ubuntu VM on Microsoft Azure using only the Azure CLI — no portal, no GUI, full automation.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Environment Details](#environment-details)
- [Objectives](#objectives)
- [Architecture and High-Level Logic](#architecture-and-high-level-logic)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Verify CLI Version and Active Session](#step-1-verify-cli-version-and-active-session)
  - [Step 2: Verify Pre-Provisioned Resource Group](#step-2-verify-pre-provisioned-resource-group)
  - [Step 3: Provision VM via Azure CLI](#step-3-provision-vm-via-azure-cli)
  - [Step 4: Confirm VM Power State](#step-4-confirm-vm-power-state)
- [Outcome](#outcome)
- [Key Azure Concepts Demonstrated](#key-azure-concepts-demonstrated)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This project documents the end-to-end provisioning of a Microsoft Azure Virtual Machine using exclusively the **Azure CLI** (`az`). The entire workflow — from environment validation to VM state confirmation — is executed programmatically, demonstrating reproducible, auditable infrastructure deployment without any reliance on the Azure Portal.

This approach aligns with modern infrastructure engineering standards: scripted, version-controllable, and portable across environments.

---

## Problem Statement

Provisioning cloud virtual machines through a graphical portal introduces human error, is difficult to audit, and cannot be reliably automated or reproduced across environments. In enterprise and CI/CD pipelines, VM provisioning must be:

- **Repeatable** across staging, UAT, and production environments
- **Auditable** through command history and script-based execution
- **Automated** without manual GUI interaction

This implementation solves that by provisioning a fully operational Ubuntu 22.04 VM using a single `az vm create` invocation, with SSH authentication auto-configured and disk parameters explicitly controlled.

---

## Environment Details

| Parameter | Value |
|---|---|
| Cloud Provider | Microsoft Azure |
| Service | Azure Virtual Machines |
| Region | `eastus` |
| Resource Group | `kml_rg_main-664660377d9743b3` |
| VM Name | `xfusion-vm` |
| OS Image | Ubuntu 22.04 LTS |
| VM Size | `Standard_B2s` |
| OS Disk Size | 30 GB |
| Storage SKU | `Standard_LRS` |
| Admin Username | `azureuser` |
| Authentication | SSH key (auto-generated) |
| CLI Version | Azure CLI 2.67.0 |

---

## Objectives

- Provision an Azure VM entirely through CLI automation
- Auto-generate SSH key pairs for secure, passwordless authentication
- Constrain OS disk size and storage SKU to lab-safe values
- Confirm VM operational state through both instance-view and list queries

---

## Architecture and High-Level Logic

```
[Azure CLI Session]
       |
       v
[Validate CLI Version + Active Account]
       |
       v
[Verify Pre-Provisioned Resource Group]
       |
       v
[az vm create]
  - Image: Ubuntu 22.04
  - Size: Standard_B2s
  - Auth: SSH (auto-generated)
  - Disk: 30GB Standard_LRS
       |
       v
[VM Provisioned → Power State: Running]
       |
       v
[Instance View Verification]
       |
       v
[VM List Confirmation: Succeeded + Running]
```

---

## Implementation Steps

### Step 1: Verify CLI Version and Active Session

Before provisioning any resource, the active CLI version and authenticated account are validated. This confirms the correct subscription context and avoids targeting the wrong tenant or environment.

```bash
az version
az account show
```

**What to verify:**
- CLI version is `2.67.0` or later
- `"isDefault": true` confirms the active subscription
- `"name": "Azure Free Labs"` confirms the target environment
- Authentication type is `servicePrincipal`, consistent with lab-provisioned access

> **Operational note:** Always run `az account show` before any destructive or provisioning operation. A misconfigured subscription context is a common source of cross-environment incidents.

**Screenshot: CLI version output and authenticated account details**

<img width="1022" height="819" alt="az version and az account show output" src="https://github.com/user-attachments/assets/46ef8a6a-d6f6-46ae-90b3-b551d42559bd" />

---

### Step 2: Verify Pre-Provisioned Resource Group

The target resource group is validated to confirm it exists, is in the correct region, and provisioning has succeeded before any compute resources are deployed into it.

```bash
az group list --output table
```

**Expected output:**

| Name | Location | Status |
|---|---|---|
| kml_rg_main-664660377d9743b3 | eastus | Succeeded |

> **Key consideration:** In lab environments, resource groups are pre-created and must not be recreated. Attempting `az group create` when the group already exists will either fail or produce a no-op depending on the idempotency behavior. Always confirm existence first.

**Screenshot: Resource group listing confirming availability in `eastus`**

<img width="1030" height="698" alt="az group list output showing resource group in Succeeded state" src="https://github.com/user-attachments/assets/cd85dce5-746d-4461-864d-ab3b3212b120" />

---

### Step 3: Provision VM via Azure CLI

With the session and resource group validated, the VM is provisioned using a single `az vm create` command with all parameters explicitly defined. No defaults are relied upon for disk or storage, ensuring reproducibility and cost control.

```bash
az vm create \
  --resource-group kml_rg_main-664660377d9743b3 \
  --name xfusion-vm \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys \
  --os-disk-size-gb 30 \
  --storage-sku Standard_LRS
```

**Parameter rationale:**

| Flag | Value | Rationale |
|---|---|---|
| `--image` | `Ubuntu2204` | LTS image, widely supported, production-suitable |
| `--size` | `Standard_B2s` | Burstable, cost-efficient for non-sustained workloads |
| `--generate-ssh-keys` | (auto) | Creates key pair in `~/.ssh/` if not present; passwordless auth |
| `--os-disk-size-gb` | `30` | Minimum viable OS disk; avoids unnecessary storage cost |
| `--storage-sku` | `Standard_LRS` | Locally redundant; appropriate for non-production and lab use |

**Successful provisioning response confirms:**
- `"powerState": "VM running"` — VM is immediately operational
- `"privateIpAddress": "10.0.0.4"` — Assigned within the default VNet
- `"publicIpAddress": "20.121.196.89"` — Reachable externally
- `"provisioningState"` embedded in resource ID confirms full ARM deployment

> **Risk:** If orphaned NICs, public IPs, or disks from a prior failed run exist, `az vm create` may error on resource conflicts. Run `az network nic list`, `az network public-ip list`, and `az disk list` to clear any stale resources before retrying.

**Screenshot: Full `az vm create` command and successful provisioning response**

<img width="1023" height="482" alt="az vm create command with all parameters and JSON response showing VM running" src="https://github.com/user-attachments/assets/2ca2f465-e217-4156-888d-c85acaa10fa2" />

---

### Step 4: Confirm VM Power State

Two verification queries are executed to confirm the VM is fully operational post-provisioning. The first targets the instance-level power status; the second lists all VMs in the resource group with full provisioning and power state details.

**Query 1: Instance view — power state only**

```bash
az vm get-instance-view \
  --name xfusion-vm \
  --resource-group kml_rg_main-664660377d9743b3 \
  --query "instanceView.statuses[1].displayStatus" \
  --output table
```

**Expected result:**

```
Result
----------
VM running
```

> **Why `statuses[1]`?** Azure instance view returns two status entries: `statuses[0]` is the provisioning state (`ProvisioningState/succeeded`), and `statuses[1]` is the power state (`PowerState/running`). Targeting index `[1]` isolates the runtime health of the VM.

**Screenshot: Instance view query returning `VM running` power state**

<img width="1035" height="475" alt="az vm get-instance-view output confirming VM running status" src="https://github.com/user-attachments/assets/29aec005-52ec-41e8-8068-e6707a59c81e" />

---

**Query 2: VM list — full provisioning and power state**

```bash
az vm list \
  --resource-group kml_rg_main-664660377d9743b3 \
  --show-details \
  --query "[].{Name:name, ProvisioningState:provisioningState, PowerState:powerState}" \
  --output table
```

**Expected result:**

| Name | ProvisioningState | PowerState |
|---|---|---|
| xfusion-vm | Succeeded | VM running |

This confirms both the ARM-level provisioning success and the live runtime state of the VM in a single command.

**Screenshot: VM list output confirming `Succeeded` provisioning state and `VM running` power state**

<img width="1041" height="674" alt="az vm list with show-details confirming xfusion-vm is Succeeded and running" src="https://github.com/user-attachments/assets/f61892e7-b59f-4eb2-b17d-823f3b09b2c0" />

---

## Outcome

| Objective | Result |
|---|---|
| VM provisioned via CLI only | Confirmed |
| SSH authentication configured | Auto-generated key pair in `~/.ssh/` |
| VM operational post-provisioning | `powerState: VM running` |
| No portal interaction required | Confirmed |
| Disk and storage parameters explicitly defined | 30 GB, `Standard_LRS` |

---

## Key Azure Concepts Demonstrated

- **Azure CLI automation** for end-to-end VM lifecycle management
- **ARM resource provisioning** via `az vm create` with explicit parameter control
- **SSH key-pair authentication** using `--generate-ssh-keys` for secure, passwordless access
- **Instance view querying** to distinguish ARM provisioning state from runtime power state
- **JMESPath query filtering** with `--query` for structured, actionable CLI output
- **Resource group scoping** to isolate and target compute resources precisely

---

## Best Practices and Operational Considerations

- **Always validate subscription context** with `az account show` before provisioning. A wrong subscription targets the wrong billing entity and potentially the wrong environment.
- **Explicitly define disk size and storage SKU.** Relying on defaults can result in oversized disks (`Premium_LRS` where `Standard_LRS` suffices), inflating cost.
- **Use `--generate-ssh-keys` over password auth.** SSH keys are non-interceptable over the wire and enforce least-privilege access patterns.
- **Check for orphaned resources** (NICs, public IPs, disks) before retrying a failed VM creation. Azure does not automatically clean up partial deployments.
- **Confirm both provisioning state and power state** post-creation. A VM can show `ProvisioningState: Succeeded` at the ARM layer while still initializing its power state. Use `get-instance-view` for the runtime health signal.
- **Prefer `--output table`** for human-readable verification steps in documentation and runbooks; prefer `--output json` when piping output to scripts or automation tools.

---

## Lessons Learned

- The `az vm create` command provisions the full network stack (VNet, subnet, NIC, public IP) automatically when no existing network resources are specified. This is convenient for isolated environments but should be explicitly controlled in shared VNets to avoid unintended topology changes.
- `statuses[1]` in the instance view is not guaranteed to be the power state across all VM configurations. In hardened environments, always filter by `code` prefix (`PowerState/`) rather than array index for reliable automation.
- SSH keys generated by `--generate-ssh-keys` are stored at `~/.ssh/id_rsa` and `~/.ssh/id_rsa.pub`. In shared or ephemeral environments, ensure these keys are backed up or rotated post-provisioning.
- `Standard_B2s` is a burstable VM size that accumulates CPU credits during idle periods and spends them under load. It is optimal for non-sustained workloads like this deployment, but should not be used for latency-sensitive production compute.

---

## Tags

`azure` `virtual-machines` `compute` `azure-cli` `infrastructure-automation` `ssh-authentication` `cloud-infrastructure` `arm` `devops` `ubuntu`



