# [Azure VM Compute Optimization: Live SKU Resize via Azure CLI](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> **Project Type:** Azure Compute | **Service:** Virtual Machines | **Operation:** VM SKU Resize | **Region:** `centralus`

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Project Objectives](#project-objectives)
- [Environment Scope and Constraints](#environment-scope-and-constraints)
- [Prerequisites](#prerequisites)
- [Step-by-Step Implementation](#step-by-step-implementation)
  - [Step 1: Retrieve Azure Credentials](#step-1-retrieve-azure-credentials)
  - [Step 2: List Available Virtual Machines](#step-2-list-available-virtual-machines)
  - [Step 3: Verify Current VM Size](#step-3-verify-current-vm-size)
  - [Step 4: Resize the Virtual Machine](#step-4-resize-the-virtual-machine)
  - [Step 5: Start the Virtual Machine](#step-5-start-the-virtual-machine)
  - [Step 6: Verify Final VM Size and State](#step-6-verify-final-vm-size-and-state)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Validation Checklist](#validation-checklist)
- [Outcome](#outcome)
- [Lessons Learned](#lessons-learned)

---

## Overview

This project documents the live resizing of an Azure Virtual Machine using the Azure CLI. The operation targets an underutilized compute resource and upgrades its SKU to a larger instance size, demonstrating a core pattern in cloud compute optimization: identifying misaligned resources, applying a non-destructive change, and validating the resulting state without downtime.

The entire workflow is executed programmatically through the Azure CLI, reinforcing Infrastructure-as-Operations principles and reproducibility without reliance on the Azure Portal UI.

---

## Problem Statement

A running Azure Virtual Machine (`xfusion-vm`) provisioned in the `centralus` region was identified as operating below its required capacity threshold. The current SKU (`Standard_B1s`) provides 1 vCPU and 1 GiB of RAM, which was insufficient for anticipated workload demands.

The task required:
- Verifying the existing compute configuration
- Upgrading the VM to `Standard_B2s` (2 vCPUs, 4 GiB RAM)
- Confirming the VM remained operational post-resize

This scenario mirrors real-world capacity planning responses where right-sizing is performed without reprovisioning the entire VM.

---

## Project Objectives

- Identify an existing Azure Virtual Machine within a target resource group
- Verify the current VM SKU via CLI query
- Resize the VM to a larger compute SKU without reprovisioning
- Ensure the VM is in a running state after the resize operation
- Confirm the final VM size and power state programmatically

---

## Environment Scope and Constraints

| Parameter | Value |
|---|---|
| **Subscription** | Azure Free Lab Environment |
| **Resource Group** | `KML_RG_MAIN-9642FE6E452040F1` |
| **Virtual Machine** | `xfusion-vm` |
| **Location** | `centralus` |
| **Original SKU** | `Standard_B1s` |
| **Target SKU** | `Standard_B2s` |
| **OS** | Ubuntu Server 22.04 LTS (Jammy) |
| **OS Disk** | 30 GB, Standard LRS |
| **Security Type** | Trusted Launch (Secure Boot + vTPM enabled) |

**Environment constraints:**
- Resource group was pre-created; no `az group create` was performed
- Storage SKU locked to `Standard_LRS`
- OS disk size set to 30 GB per lab policy
- VM size constrained to B-series SKUs available in `centralus`

---

## Prerequisites

- Azure CLI installed and authenticated (`az login` or service principal)
- Active Azure subscription with Virtual Machine Contributor permissions
- Target VM (`xfusion-vm`) provisioned and accessible in the resource group
- Network and disk resources in a clean, non-conflicting state

---

## Step-by-Step Implementation

### Step 1: Retrieve Azure Credentials

The `showcreds` utility exposes the lab-provisioned Azure credentials, including the portal URL, username, password, application client ID, and session expiry time. This establishes the authenticated context for all subsequent CLI operations.

**Command:**
```bash
showcreds
```

**Expected Output:**
- Azure Console URL
- Azure User Name
- Azure Password (masked)
- Azure Application Client ID
- Azure Session End Time

![Step 1 - Retrieve Azure lab credentials using showcreds](https://github.com/user-attachments/assets/a6d7b673-18a2-4a40-a879-443502dfc12b)
*Screenshot: `showcreds` output displaying the Azure portal URL, lab user credentials, application client ID, and session expiry timestamp.*

---

### Step 2: List Available Virtual Machines

Before performing any modification, enumerate all VMs in the subscription to confirm the target VM exists and identify its resource group and region. This prevents operating on the wrong resource and establishes the correct resource group reference.

**Command:**
```bash
az vm list \
  --query "[].{VM_Name:name, ResourceGroup:resourceGroup, Location:location}" \
  -o table
```

**Expected Output:**

| VM_Name | ResourceGroup | Location |
|---|---|---|
| xfusion-vm | KML_RG_MAIN-9642FE6E452040F1 | centralus |

![Step 2 - List all VMs in the subscription with name, resource group, and location](https://github.com/user-attachments/assets/09466e95-8808-4ab4-bc4e-3a2f9ca970b0)
*Screenshot: `az vm list` output confirming `xfusion-vm` is present in resource group `KML_RG_MAIN-9642FE6E452040F1` in the `centralus` region.*

**Operational note:** Always verify VM existence and location before executing lifecycle operations. Misidentified resource groups are a common source of unintended changes in multi-environment subscriptions.

---

### Step 3: Verify Current VM Size

Query the VM hardware profile to retrieve the current SKU. This serves as both a pre-change baseline and a validation anchor for confirming the resize delta in Step 6.

**Command:**
```bash
az vm show \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm \
  --query hardwareProfile.vmSize
```

**Expected Output:**
```
"Standard_B1s"
```

![Step 3 - Query the current VM hardware profile to confirm the active SKU](https://github.com/user-attachments/assets/a67aa12e-9d6a-44a2-826a-66287742fb54)
*Screenshot: `az vm show` confirming the current VM size is `Standard_B1s` prior to the resize operation.*

**Key Decisions:**
- `--query hardwareProfile.vmSize` targets the specific field rather than returning the full VM object, reducing noise and improving auditability in CI/CD or scripted pipelines.
- Capturing the current SKU before modification is a prerequisite for change documentation and rollback planning.

---

### Step 4: Resize the Virtual Machine

Apply the new compute SKU to the VM using `az vm update` with the `--size` flag. This operation modifies the VM hardware profile and updates the Azure Compute resource model. The VM may undergo a brief restart depending on host capacity.

**Command:**
```bash
az vm update \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm \
  --size Standard_B2s
```

**System Note:**
```
Argument '--size' is in preview and under development.
Reference and support levels: https://aka.ms/CLI_refstatus
```

> **Warning:** The `--size` parameter in `az vm update` is a preview feature as of the time of this operation. For production environments, consider using `az vm resize` as the stable alternative. Both achieve the same outcome, but preview features may have breaking changes across CLI versions.

**Expected Output (key fields):**

```json
{
  "hardwareProfile": {
    "vmSize": "Standard_B2s"
  },
  "provisioningState": "Succeeded",
  "name": "xfusion-vm",
  "location": "centralus",
  "resourceGroup": "KML_RG_MAIN-9642FE6E452040F1"
}
```

![Step 4a - Initiating the VM resize and observing the updated hardware profile in the JSON response](https://github.com/user-attachments/assets/3115eb1a-2257-40d4-a724-86617c7551e7)
*Screenshot: `az vm update` command execution, showing the preview flag warning and the start of the full VM JSON response.*

![Step 4b - JSON response confirming vmSize updated to Standard_B2s and osProfile details](https://github.com/user-attachments/assets/93424d0b-a961-487a-b292-554a03f0bcb2)
*Screenshot: Continued JSON output showing `osProfile` configuration including `adminUsername: azureuser`, SSH key path, and `provisionVmAgent: true`.*

![Step 4c - Storage profile and security profile confirming TrustedLaunch, Secure Boot, and vTPM](https://github.com/user-attachments/assets/9e8c714c-12d8-48e8-a0b5-6c623a16f935)
*Screenshot: Storage and security profile blocks confirming `Standard_B2s` resize, `securityType: TrustedLaunch`, `secureBootEnabled: true`, and `vTpmEnabled: true`.*

![Step 4d - Final JSON block confirming provisioningState Succeeded and VM metadata](https://github.com/user-attachments/assets/d5abc90e-4c9a-4575-a1ca-f1545543fae4)
*Screenshot: JSON tail confirming `provisioningState: Succeeded`, `timeCreated`, `vmId`, and closure of the `az vm update` response.*

**Key Decisions:**
- `Standard_B2s` was selected over `Standard_B2ms` because the workload profile required additional vCPUs but not the elevated memory of the memory-optimized variant.
- The `--size` preview flag was accepted given the non-production lab context. In production, use `az vm resize` with pre-validated SKU availability via `az vm list-vm-resize-options`.

---

### Step 5: Start the Virtual Machine

After a resize operation, the VM may be in a deallocated or stopped state depending on whether the target SKU required a host migration. Issue an explicit start command to bring the VM back to a running state.

**Command:**
```bash
az vm start \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm
```

**Expected Result:**
- VM transitions from deallocated or stopped state to `VM running`
- Command exits with no errors upon successful power-on

![Step 5 - Starting the VM after resize to restore it to running state](https://github.com/user-attachments/assets/38a6cbf7-df18-4384-89e8-8da11e630de8)
*Screenshot: `az vm start` command execution followed immediately by the final `az vm get-instance-view` validation query, confirming the VM returned to a running state.*

**Operational note:** Always issue `az vm start` after a resize in automated pipelines. Do not assume the VM retains its power state post-resize, particularly when the operation results in host migration due to SKU family changes.

---

### Step 6: Verify Final VM Size and State

Query the VM instance view to simultaneously confirm both the active hardware SKU and the current power state. This dual-field validation is the authoritative check that the resize and start sequence completed successfully.

**Command:**
```bash
az vm get-instance-view \
  --resource-group KML_RG_MAIN-9642FE6E452040F1 \
  --name xfusion-vm \
  --query "{Size:hardwareProfile.vmSize, State:instanceView.statuses[1].displayStatus}"
```

**Expected Output:**
```json
{
  "Size": "Standard_B2s",
  "State": "VM running"
}
```

![Step 6 - Final validation confirming VM size is Standard_B2s and power state is VM running](https://github.com/user-attachments/assets/90502930-4c46-4749-9edb-f7f20fc9ef99)
*Screenshot: `az vm get-instance-view` output confirming `Size: Standard_B2s` and `State: VM running`, completing the resize validation.*

**Key Decisions:**
- `instanceView.statuses[1].displayStatus` targets the power state status. Index `[0]` captures the provisioning state; index `[1]` captures the actual runtime power state. Using the correct index avoids misreading `ProvisioningState/succeeded` as a running VM.
- The combined `{Size, State}` JMESPath projection reduces output verbosity and makes the validation output machine-parseable for pipeline assertions.

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Used `az vm update --size` (preview) | Faster single-command approach acceptable in a non-production environment |
| Chose `Standard_B2s` over `Standard_B2ms` | Workload required additional vCPUs, not additional memory |
| Queried `hardwareProfile.vmSize` before and after | Establishes a documented delta for change auditing and rollback verification |
| Used `instanceView.statuses[1]` for power state | Index `[0]` is provisioning state; index `[1]` is the actual VM power state |
| Issued explicit `az vm start` post-resize | VM power state is not guaranteed to be preserved after a resize with host migration |

---

## Errors and Resolutions

**Preview flag warning during `az vm update --size`**

- **Symptom:** CLI emits `Argument '--size' is in preview and under development` before executing.
- **Impact:** None on execution; the operation completes successfully.
- **Resolution:** Accepted in the lab context. For production pipelines, replace with `az vm resize --size Standard_B2s` which is the stable command for SKU changes.

**VM not running after resize**

- **Symptom:** VM enters a deallocated state after SKU update, particularly when Azure migrates the VM to a different host to support the new SKU family.
- **Impact:** Applications running on the VM are interrupted until an explicit start is issued.
- **Resolution:** Always follow a resize operation with `az vm start` and validate state via `az vm get-instance-view`.

---

## Best Practices and Operational Considerations

- **Pre-validate SKU availability** before resizing in production. Run `az vm list-vm-resize-options` to confirm the target SKU is available in the current cluster and host. Attempting to resize to an unavailable SKU results in an allocation failure that may require deallocating the VM first.
- **Prefer `az vm resize` over `az vm update --size`** in production and CI/CD pipelines. The former is the stable, purpose-built command; the latter is a preview feature subject to change.
- **Capture VM state before and after** in change records. Logging both `az vm show --query hardwareProfile.vmSize` outputs before and after provides an auditable trail for change management processes.
- **Account for restart impact** in maintenance windows. Depending on the SKU family change, Azure may require a full VM deallocation and reallocation to a new host. Plan for brief unavailability in production.
- **Use `--no-wait` with monitoring for long-running resize operations** in automated workflows to avoid blocking pipeline execution, then poll with `az vm get-instance-view` until the desired state is confirmed.
- **Tag VMs with intended SKU and sizing rationale** using Azure resource tags to enable cost governance reviews and prevent unintended size drift over time.

---

## Validation Checklist

- [x] VM `xfusion-vm` exists in `centralus` region under `KML_RG_MAIN-9642FE6E452040F1`
- [x] Original VM size confirmed as `Standard_B1s` before modification
- [x] VM resized to `Standard_B2s` with `provisioningState: Succeeded`
- [x] VM started successfully after resize
- [x] Final VM size confirmed as `Standard_B2s` via `az vm get-instance-view`
- [x] Final VM power state confirmed as `VM running`

---

## Outcome

The Virtual Machine `xfusion-vm` was successfully resized from `Standard_B1s` to `Standard_B2s` using the Azure CLI. The operation updated the VM hardware profile, was provisioned with a result of `Succeeded`, and the VM was explicitly started and confirmed operational post-resize. All validation checks passed, meeting all defined project requirements.

---

## Lessons Learned

- **`az vm update --size` is a preview feature.** While functional, it should not be used in production without acknowledging the maturity caveat. The stable equivalent is `az vm resize`.
- **VM power state is not guaranteed after a resize.** Always treat the post-resize power state as unknown and issue an explicit `az vm start` followed by a state check.
- **JMESPath index matters when querying instance statuses.** `statuses[0]` returns provisioning state, not power state. `statuses[1]` is the correct target for confirming `VM running`.
- **SKU availability must be validated before production resizes.** Not all SKUs are available in all regions and availability zones. Failing to check can result in allocation errors that require full VM deallocation to recover from.
- **Programmatic validation closes the loop.** Using `az vm get-instance-view` with a structured JMESPath projection provides a deterministic, scriptable verification step that can be gated in CI/CD pipelines.
