# Attaching a Network Interface to an Azure Virtual Machine via Azure CLI

> **Production-Style Infrastructure Operations Guide**
> Authored for enterprise environments, onboarding engineers, and production-level handoff documentation.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Infrastructure Details](#infrastructure-details)
- [Prerequisites](#prerequisites)
- [Architecture Context](#architecture-context)
- [Implementation: Step-by-Step](#implementation-step-by-step)
  - [Step 1: Identify the Resource Group](#step-1-identify-the-resource-group)
  - [Step 2: Deallocate the Virtual Machine](#step-2-deallocate-the-virtual-machine)
  - [Step 3: Attach the Network Interface to the VM](#step-3-attach-the-network-interface-to-the-vm)
  - [Step 4: Start the Virtual Machine](#step-4-start-the-virtual-machine)
  - [Step 5: Verify VM Power State](#step-5-verify-vm-power-state)
- [Validation Checklist](#validation-checklist)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Operational Best Practices](#operational-best-practices)
- [Outcome and Lessons Learned](#outcome-and-lessons-learned)
- [Tags](#tags)

---

## Overview

This document details the end-to-end process for attaching an existing **Azure Network Interface Card (NIC)** to a running **Azure Virtual Machine (VM)** using the **Azure CLI (`az`)**. It follows a structured problem-solution-implementation-validation model appropriate for production systems and engineering handoff.

---

## Problem Statement

In Azure infrastructure, there are scenarios where a VM requires an additional network interface, for example:

- Enabling **network traffic segmentation** across subnets
- Supporting **dual-homed** architectures (e.g., frontend and backend interface separation)
- Recovering a misconfigured primary NIC by attaching a pre-configured secondary NIC
- Scaling network throughput by introducing additional NICs

Azure does **not** permit attaching or detaching NICs from a running VM. The VM must first be deallocated, the NIC change applied, and the VM restarted. This guide walks through each step of that process precisely as executed in a live Azure environment.

---

## Infrastructure Details

| Property | Value |
|---|---|
| **Cloud Provider** | Microsoft Azure |
| **Region** | `eastus` |
| **Resource Group** | `kml_rg_main-ae708aa3b65d4be6` |
| **Virtual Machine** | `nautilus-vm` |
| **Network Interface** | `nautilus-nic` |
| **CLI Tool** | Azure CLI (`az`) |

---

## Prerequisites

Before beginning, ensure the following conditions are satisfied:

- **Azure CLI** is installed and authenticated (`az login` completed successfully)
- The **target VM** (`nautilus-vm`) exists within the target resource group
- The **NIC** (`nautilus-nic`) exists within the **same resource group** as the VM
- The NIC is **not already attached** to another VM
- The NIC's **subnet** is compatible with the VM's virtual network
- Operator has **Contributor** or higher RBAC permissions on the resource group
- Planned **maintenance window** is in effect if this is a production VM (deallocating stops the VM)

---

## Architecture Context

Azure Virtual Machines support multiple NICs depending on the VM size/SKU. Each NIC is an independent Azure resource that can be created, attached, detached, and reused independently of any VM lifecycle.

**Key constraints:**
- A VM must be in a **deallocated** (stopped) state before NIC changes can be applied
- The **primary NIC** cannot be removed while secondary NICs are present
- All NICs attached to a VM must reside in the **same virtual network**, though they may be in different subnets
- NIC attachment order determines which NIC is primary; the first NIC attached at VM creation is always primary

---

## Implementation: Step-by-Step

### Step 1: Identify the Resource Group

Before performing any operations, confirm the correct Azure Resource Group name. This prevents accidental modifications to unintended environments.

**Command:**
```bash
az group list --query "[].name" -o tsv
```

**Output:**
```
kml_rg_main-ae708aa3b65d4be6
```

**Screenshot: Listing Azure Resource Groups**

> Confirms the active resource group `kml_rg_main-ae708aa3b65d4be6` is available in the authenticated subscription.

![az-group-list](https://github.com/user-attachments/assets/e72f0ede-4262-4b54-915f-2b9d59b27314)

**Operational Note:** If multiple resource groups are returned, use `--query "[?contains(name, 'your-prefix')].name"` to filter results. Always confirm the subscription context with `az account show` before proceeding.

---

### Step 2: Deallocate the Virtual Machine

Azure requires the VM to be in a fully **deallocated** state before NIC configuration changes can be made. A simple "stop" (OS shutdown) is not sufficient; the VM must be deallocated to release the underlying compute fabric.

**Command:**
```bash
az vm deallocate \
  --resource-group kml_rg_main-ae708aa3b65d4be6 \
  --name nautilus-vm
```

**Screenshot: Deallocating the Virtual Machine**

> The CLI command completes without error, indicating the VM has been successfully deallocated and is ready for NIC modification.

![vm-deallocated](https://github.com/user-attachments/assets/ab6d4de4-31c7-450e-b110-1ae142658f99)

**Operational Notes:**
- Deallocation **releases the public IP** if it is dynamically assigned. Use a **static public IP** in production to avoid IP address changes on restart.
- This operation causes **downtime** for any services running on `nautilus-vm`. Coordinate with stakeholders and schedule within a maintenance window.
- Deallocation also **stops billing** for compute (vCPU and RAM), though storage charges continue.
- Verify deallocation status with: `az vm show -d -g kml_rg_main-ae708aa3b65d4be6 -n nautilus-vm --query "powerState"`

---

### Step 3: Attach the Network Interface to the VM

With the VM deallocated, attach the pre-existing NIC (`nautilus-nic`) to the VM (`nautilus-vm`) using the `az vm nic add` command.

**Command:**
```bash
az vm nic add \
  --resource-group kml_rg_main-ae708aa3b65d4be6 \
  --vm-name nautilus-vm \
  --nics nautilus-nic
```

**Expected Behavior:**
- The existing **primary NIC** (`nautilus-vmVMNic`) is retained with `"primary": true`
- The newly attached NIC (`nautilus-nic`) appears with `"primary": false`
- No CLI errors are returned

**Output (JSON):**
```json
[
  {
    "deleteOption": null,
    "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-ae708aa3b65d4be6/providers/Microsoft.Network/networkInterfaces/nautilus-vmVMNic",
    "primary": true,
    "resourceGroup": "kml_rg_main-ae708aa3b65d4be6"
  },
  {
    "deleteOption": null,
    "id": "/subscriptions/f0c3bcdd-5ce2-4fa0-8cf3-41559747512b/resourceGroups/kml_rg_main-ae708aa3b65d4be6/providers/Microsoft.Network/networkInterfaces/nautilus-nic",
    "primary": false,
    "resourceGroup": "kml_rg_main-ae708aa3b65d4be6"
  }
]
```

**Screenshot: NIC Successfully Attached to VM (CLI Output)**

> The JSON response confirms two NICs are now associated with `nautilus-vm`. The original primary NIC (`nautilus-vmVMNic`) remains primary; `nautilus-nic` is attached as the secondary interface.

![nic-attached-cli-output](https://github.com/user-attachments/assets/ac4e0c9e-215f-44df-bf03-95f118088ebf)

**Operational Notes:**
- Multiple NICs can be specified in a single command by space-separating them: `--nics nic1 nic2`
- The primary NIC is determined at VM creation and **cannot be changed** without rebuilding the VM
- Confirm the NIC's subnet is within the same VNet as the primary NIC before attaching

---

### Step 4: Start the Virtual Machine

After the NIC has been successfully attached, restart the VM to bring it back into a running state.

**Command:**
```bash
az vm start \
  --resource-group kml_rg_main-ae708aa3b65d4be6 \
  --name nautilus-vm
```

**Screenshot: Starting the Virtual Machine**

> The CLI command completes successfully with no errors, confirming the VM boot sequence has been initiated.

![vm-started](https://github.com/user-attachments/assets/f5602539-adee-4844-acd3-c3632b26bbbf)

**Operational Notes:**
- VM boot may take 1-3 minutes depending on the OS and startup configuration
- If the VM has a **cloud-init** or **custom script extension**, additional time may be required before the VM is fully ready for traffic
- For critical services, validate application health endpoints after start, not just VM power state

---

### Step 5: Verify VM Power State

After starting the VM, programmatically confirm it has reached the `running` power state before declaring the operation complete.

**Command:**
```bash
az vm show -d \
  -g kml_rg_main-ae708aa3b65d4be6 \
  -n nautilus-vm \
  --query "powerState"
```

**Expected Output:**
```
"VM running"
```

**Screenshot: VM Power State Confirmed as Running**

> The `powerState` query returns `"VM running"`, confirming the VM has successfully restarted with the newly attached NIC active.

![vm-running-status](https://github.com/user-attachments/assets/3b53d975-7360-4bef-add4-218ca705fc83)

**Operational Notes:**
- `"VM running"` is the only acceptable terminal state for this operation. Any other value (e.g., `"VM starting"`, `"VM stopped"`) indicates an incomplete or failed operation.
- For deeper NIC validation, confirm interface visibility within the VM OS using `ip addr` (Linux) or `ipconfig` (Windows) via SSH/RDP or Azure Run Command:
  ```bash
  az vm run-command invoke \
    --resource-group kml_rg_main-ae708aa3b65d4be6 \
    --name nautilus-vm \
    --command-id RunShellScript \
    --scripts "ip addr show"
  ```

---

## Validation Checklist

Use this checklist to confirm a successful, production-ready NIC attachment:

- [ ] Resource group correctly identified (`kml_rg_main-ae708aa3b65d4be6`)
- [ ] VM successfully deallocated before NIC modification
- [ ] `nautilus-nic` attached as secondary NIC without CLI errors
- [ ] Primary NIC (`nautilus-vmVMNic`) remains unchanged with `"primary": true`
- [ ] VM started without errors
- [ ] VM power state confirmed as `"VM running"`
- [ ] Secondary NIC visible within the VM OS (via `ip addr` or `ipconfig`)
- [ ] Network connectivity verified over the new NIC (if applicable)
- [ ] Static IP / DNS records updated if necessary

---

## Risks, Edge Cases, and Troubleshooting

| Scenario | Risk | Resolution |
|---|---|---|
| VM has dynamic public IP | IP address changes on restart | Assign a **static public IP** before deallocation |
| NIC in different VNet | Attachment will fail | Ensure NIC and VM share the same Virtual Network |
| NIC already attached to another VM | CLI error: NIC in use | Detach from current VM before reattaching |
| VM does not support multiple NICs | CLI error: VM SKU limit | Resize VM to a SKU that supports multiple NICs |
| VM fails to start after NIC attachment | Boot failure or timeout | Check Azure Activity Log and VM boot diagnostics |
| OS does not auto-configure new NIC | NIC not visible inside VM | Manually configure interface via `ip addr` or DHCP restart |
| Deallocation times out | Long-running deallocation | Check for VM extensions or OS-level locks; force deallocate if needed |

---

## Operational Best Practices

- **Always deallocate, never just stop.** An OS-level shutdown is insufficient for NIC changes in Azure. Use `az vm deallocate`.
- **Use static IP addressing** for production VMs to prevent IP churn across deallocate-start cycles.
- **Tag resources consistently.** Ensure NICs follow the same naming and tagging conventions as their parent VM for traceability.
- **Audit with Activity Logs.** After any infrastructure change, verify the operation in the Azure Activity Log (`az monitor activity-log list`) to maintain a clear audit trail.
- **Automate with scripts or IaC.** For repeatable NIC attachment operations, encode this workflow in an Azure CLI script, Bicep template, or Terraform module to reduce manual error.
- **Test NIC connectivity post-attachment.** Validate end-to-end network reachability via the new NIC before closing the change record.
- **Use `--no-wait` for non-blocking operations in automation pipelines**, and poll status separately to avoid timeouts.

---

## Outcome and Lessons Learned

**Result:** The existing network interface `nautilus-nic` was successfully attached to `nautilus-vm` as a secondary NIC using Azure CLI. The VM was confirmed running with both NICs active post-restart.

**Key Takeaways:**

- Azure mandates a **deallocate-modify-start** cycle for any NIC configuration change; there is no live-attach capability
- The **primary NIC is immutable** once the VM is provisioned; plan NIC topology at VM creation time
- A clean JSON response from `az vm nic add` with both NICs listed confirms successful attachment before the VM is restarted
- Automating the `powerState` check post-start provides a reliable, scriptable gate for pipeline-based deployments

---

## Tags

`azure` `azure-cli` `networking` `virtual-machines` `nic` `cloud-operations` `infrastructure` `devops` `eastus`
