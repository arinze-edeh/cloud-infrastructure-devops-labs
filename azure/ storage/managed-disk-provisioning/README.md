# Azure Managed Disk Provisioning via CLI: Standard_LRS for Incremental Cloud Migration

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Objectives](#objectives)
- [Tools and Services](#tools-and-services)
- [Preconditions](#preconditions)
- [Implementation](#implementation)
  - [Step 1: Verify Azure Account Context](#step-1-verify-azure-account-context)
  - [Step 2: Identify Target Resource Group](#step-2-identify-target-resource-group)
  - [Step 3: Create the Managed Disk](#step-3-create-the-managed-disk)
  - [Step 4: Verify Disk Creation](#step-4-verify-disk-creation)
- [Validation Checklist](#validation-checklist)
- [Errors and Resolutions](#errors-and-resolutions)
- [Outcome](#outcome)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Lessons Learned](#lessons-learned)

---

## Overview

This document details the provisioning of an Azure Managed Disk using the Azure CLI as part of an incremental cloud storage migration initiative. The approach prioritizes automation-readiness, auditability, and alignment with DevOps and SRE standards for production infrastructure management.

---

## Problem Statement

During phased cloud migrations, storage components must be provisioned independently of compute to allow teams to validate, test, and attach volumes to virtual machines on demand. Manually creating disks through the Azure Portal introduces inconsistency and is not repeatable at scale. The goal is to establish a CLI-driven, scriptable workflow for managed disk creation that can be integrated into CI/CD pipelines and Infrastructure-as-Code (IaC) processes.

---

## Architecture Context

Managed Disks in Azure are durable, block-level storage volumes managed by the Azure platform. They decouple storage provisioning from VM lifecycle management, enabling clean attachment and detachment workflows. In an incremental migration model, disks are pre-provisioned in the correct resource group and region before compute resources are created, reducing provisioning dependencies and enabling parallel workstream execution.

---

## Objectives

- Provision an Azure Managed Disk with the following specifications:
  - **Disk Name:** `nautilus-disk`
  - **Disk Type:** `Standard_LRS` (Locally Redundant Storage, Standard HDD)
  - **Disk Size:** `2 GiB`
  - **Disk State post-provisioning:** `Unattached`
  - **Provisioning Method:** Azure CLI

---

## Tools and Services

- **Azure CLI** - primary provisioning and verification interface
- **Azure Managed Disks** - the target storage resource
- **Azure Resource Groups** - logical container for the disk resource

---

## Preconditions

- Azure CLI installed and configured in the local or cloud shell environment
- Authenticated Azure session using a **service principal** (non-interactive, automation-compatible)
- An existing resource group available in the target subscription
- Correct subscription context active prior to provisioning

---

## Implementation

---

### Step 1: Verify Azure Account Context

Before any provisioning action, confirm the active Azure session is correctly authenticated and targeting the intended subscription and tenant. This eliminates misprovisioning caused by stale or incorrect account context.

**Command:**
```bash
az account show
```

**Validate the following fields in the output:**
- `"state": "Enabled"` - subscription is active
- `"type": "servicePrincipal"` - non-interactive authentication confirmed
- `"homeTenantId"` and `"tenantId"` match expected tenant
- `"isDefault": true` - this subscription is the active context

📸 **Screenshot: Active Subscription and Service Principal Authentication Confirmed**

<img width="1032" height="618" alt="image" src="https://github.com/user-attachments/assets/42db5718-90c9-461f-a78a-2025ac7a127d" />

> **Operational Note:** In shared or multi-subscription environments, always run `az account set --subscription <id>` before provisioning to prevent deploying resources into the wrong subscription. Service principal authentication is preferred over interactive login for audit traceability.

---

### Step 2: Identify Target Resource Group

Retrieve the list of available resource groups in the active subscription to confirm the target resource group name. This avoids typos and ensures the disk is provisioned into the correct logical container.

**Command:**
```bash
az group list --query "[].name" -o tsv
```

**Selected Resource Group:**
```
kml_rg_main-d71f6b2326b14421
```

📸 **Screenshot: Available Resource Groups Retrieved via CLI Query**

<img width="1028" height="621" alt="image" src="https://github.com/user-attachments/assets/d22205e4-6aff-482f-8496-9beef4330202" />

> **Operational Note:** The `--query "[].name"` JMESPath filter combined with `-o tsv` produces clean, script-consumable output. In environments with multiple resource groups, extend the query to filter by region or tag (e.g., `--query "[?location=='eastus'].name"`).

---

### Step 3: Create the Managed Disk

Provision the Azure Managed Disk with the specified parameters. The `az disk create` command is idempotent-friendly and returns the full resource JSON on success, enabling downstream automation to parse and act on provisioning results immediately.

**Command:**
```bash
az disk create \
  --resource-group kml_rg_main-d71f6b2326b14421 \
  --name nautilus-disk \
  --sku Standard_LRS \
  --size-gb 2
```

**Key Parameters:**

| Parameter | Value | Rationale |
|---|---|---|
| `--resource-group` | `kml_rg_main-d71f6b2326b14421` | Pre-existing container for resource logical grouping |
| `--name` | `nautilus-disk` | Descriptive, project-scoped name |
| `--sku` | `Standard_LRS` | Cost-efficient HDD storage with local redundancy; appropriate for non-production or migration staging |
| `--size-gb` | `2` | Minimum viable size for migration staging disk |

**Expected output fields to confirm:**

- `"provisioningState": "Succeeded"`
- `"diskState": "Unattached"`
- `"diskSizeGb": 2`
- `"sku.name": "Standard_LRS"`
- `"location": "eastus"`
- `"encryption.type": "EncryptionAtRestWithPlatformKey"` (default platform-managed encryption active)

📸 **Screenshot: Disk Creation Command Execution and JSON Response (Part 1)**

<img width="1025" height="858" alt="image" src="https://github.com/user-attachments/assets/f1fd5a65-6a3b-4d75-aba8-28447eee72c7" />

📸 **Screenshot: Disk Creation JSON Response (Part 2) - Provisioning State and SKU Confirmed**

<img width="1028" height="866" alt="image" src="https://github.com/user-attachments/assets/6b9e0617-3466-4451-801d-fe4970c01423" />

📸 **Screenshot: Disk Creation JSON Response (Part 3) - uniqueId, timeCreated, and Zone Fields**

<img width="1033" height="735" alt="image" src="https://github.com/user-attachments/assets/8be0f3ad-117a-45c3-b43e-7f3093de838f" />

> **Operational Note:** Platform-managed encryption (`EncryptionAtRestWithPlatformKey`) is enabled by default on all Azure Managed Disks at no additional cost. For workloads with regulatory or compliance requirements, consider Customer-Managed Keys (CMK) via Azure Key Vault.

---

### Step 4: Verify Disk Creation

After provisioning, explicitly verify the disk resource is present, correctly sized, and in a healthy state. This step provides a clean, human-readable confirmation and serves as the final gate before compute attachment.

**Command (with flag resolution):**

The abbreviated `--o table` flag caused an ambiguity error, as it matched both `--only-show-errors` and `--output`. The correct full flag `--output table` was used:

```bash
az disk show \
  --resource-group kml_rg_main-d71f6b2326b14421 \
  --name nautilus-disk \
  --output table
```

**Expected output:**

| Name | ResourceGroup | Location | Zones | Sku | SizeGb | ProvisioningState |
|---|---|---|---|---|---|---|
| nautilus-disk | kml_rg_main-d71f6b2326b14421 | eastus | | Standard_LRS | 2 | Succeeded |

📸 **Screenshot: Disk Verification - Table Output Confirming Successful Provisioning**

<img width="1030" height="474" alt="image" src="https://github.com/user-attachments/assets/15da9d71-de80-49f1-b9dd-9d74af05467b" />

---

## Validation Checklist

- **Disk name** matches specification: `nautilus-disk`
- **Disk size** is exactly `2 GiB`
- **Disk type** is `Standard_LRS`
- **Provisioning state** is `Succeeded`
- **Disk state** is `Unattached` (ready for VM attachment)
- **Location** matches target region: `eastus`
- **Encryption** at rest is active (platform-managed key)

---

## Errors and Resolutions

### Error: Ambiguous CLI Option `--o table`

**Command used:**
```bash
az disk show --resource-group kml_rg_main-d71f6b2326b14421 --name nautilus-disk --o table
```

**Error message:**
```
ambiguous option: --o could match --only-show-errors, --output
```

**Root Cause:** The Azure CLI argument parser could not resolve the abbreviated flag `--o` because it matched two distinct options: `--only-show-errors` and `--output`.

**Resolution:** Use the full flag name to eliminate ambiguity:
```bash
az disk show \
  --resource-group kml_rg_main-d71f6b2326b14421 \
  --name nautilus-disk \
  --output table
```

**Lesson:** Always use fully qualified flags in CLI commands, especially in scripts and IaC pipelines, to prevent parsing failures across CLI versions or environments.

---

## Outcome

- Azure Managed Disk `nautilus-disk` successfully provisioned via CLI in region `eastus`
- Disk is `Unattached` and immediately available for compute integration
- Storage component is staged and ready for the next phase of the incremental migration workflow
- Full provisioning audit trail captured via CLI JSON output and structured verification

---

## Key Decisions

- **`Standard_LRS` over `Premium_LRS`:** Standard HDD-backed storage was selected for this migration staging disk as the workload does not require low-latency SSD performance. Standard_LRS provides adequate throughput at significantly lower cost, appropriate for pre-production and incremental migration phases. Premium_LRS would be the correct choice for IOPS-intensive or production-critical workloads.

- **CLI over Portal:** The Azure CLI was chosen over the Portal UI to ensure the provisioning workflow is scriptable, repeatable, and pipeline-compatible. CLI commands are version-controllable, audit-friendly, and integrate directly into CI/CD toolchains.

- **2 GiB minimum size:** The smallest valid managed disk size was selected to validate provisioning mechanics without unnecessary resource cost. Disk size can be expanded post-creation using `az disk update --size-gb <new_size>` without data loss.

- **No Availability Zone specified:** The disk was provisioned without a zone assignment (`"zones": null`), making it a regional resource. For production workloads requiring high availability, pin disks to specific zones matching the target VM zone (`--zone 1`, `--zone 2`, etc.) to avoid cross-zone attachment latency.

---

## Best Practices and Operational Considerations

- **Naming conventions:** Use consistent, project-scoped disk names that reflect the workload and environment (e.g., `<project>-<env>-<region>-disk`). This simplifies resource filtering and cost allocation.
- **Tagging:** Apply resource tags at creation time using `--tags environment=staging project=nautilus`. Tags are essential for cost management, ownership tracking, and automation targeting.
- **Disk snapshots:** Before attaching a disk to a VM and writing data, create a baseline snapshot using `az snapshot create`. This provides a restore point for rollback scenarios.
- **Role-Based Access Control (RBAC):** Scope managed disk creation permissions to the minimum required role. Contributors at the resource group level are sufficient; avoid subscription-wide Contributor assignments.
- **Disk encryption:** Platform-managed encryption is enabled by default. For regulated workloads, configure Customer-Managed Keys (CMK) with Azure Key Vault and Disk Encryption Sets.
- **Service principal authentication:** Non-interactive service principal sessions align with enterprise security posture. Rotate service principal credentials regularly and scope permissions to the minimum required resource group or subscription.

---

## Lessons Learned

- **Fully qualify all CLI flags in scripts.** Abbreviated flags like `--o` are convenient interactively but introduce ambiguity in scripts and across CLI versions. Always use the full flag name (`--output`) for reliability and portability.
- **Always verify active account context before provisioning.** Running `az account show` at the start of every session prevents misprovisioning in multi-subscription environments, which can be costly and difficult to remediate.
- **Disk state matters for attachment.** A disk with `diskState: Attached` cannot be attached to another VM without first detaching it. The `Unattached` state confirmed here is the prerequisite for VM integration.
- **CLI JSON output enables downstream automation.** The full JSON response from `az disk create` contains the disk `id`, `uniqueId`, and `provisioningState`, all of which can be consumed programmatically by subsequent automation steps or piped into Azure Resource Manager deployments.
- **Incremental migration reduces operational risk.** Provisioning storage independently of compute allows for validation at each phase, isolating failure domains and enabling parallel workstream execution across infrastructure teams.


