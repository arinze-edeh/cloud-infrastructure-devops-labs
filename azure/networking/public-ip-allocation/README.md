# Azure Static Public IP Provisioning via CLI: Handling SKU Allocation Constraints

> **Domain:** Cloud Networking | Azure CLI | Infrastructure Foundation
> **Scope:** Public IP Allocation with SKU-Aware Constraint Resolution
> **Environment:** Microsoft Azure | East US Region | Standard SKU

---

## Table of Contents

- [Overview](#overview)
- [Problem Definition](#problem-definition)
- [Architecture Context](#architecture-context)
- [Prerequisites](#prerequisites)
- [High-Level Logic](#high-level-logic)
- [Implementation](#implementation)
  - [Step 1: Retrieve Azure Credentials](#step-1-retrieve-azure-credentials)
  - [Step 2: Identify the Active Resource Group](#step-2-identify-the-active-resource-group)
  - [Step 3: Attempt Public IP Creation with Dynamic Allocation](#step-3-attempt-public-ip-creation-with-dynamic-allocation)
  - [Step 4: Recreate Public IP with Static Allocation and Standard SKU](#step-4-recreate-public-ip-with-static-allocation-and-standard-sku)
  - [Step 5: Verify Public IP Provisioning](#step-5-verify-public-ip-provisioning)
- [Final Outcome](#final-outcome)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Tags](#tags)

---

## Overview

The Nautilus DevOps team is executing a phased migration of infrastructure to Microsoft Azure. As part of establishing the networking foundation, this document details the provisioning of a **Static Public IP address** (`devops-pip`) to support external connectivity for future Azure resources.

This guide walks through the complete process using the **Azure CLI**, including encountering and resolving a SKU-level allocation constraint that surfaces when attempting Dynamic allocation against a Standard SKU environment. The resolution pattern documented here is directly applicable to production Azure networking scenarios.

---

## Problem Definition

During initial Public IP provisioning, a **Standard SKU enforcement policy** prevents the use of Dynamic allocation. Specifically:

- Standard SKU Public IPs in Azure **require Static allocation**
- Attempting Dynamic allocation returns a hard error: `StandardAndStandardV2SkuPublicIPAddressesMustBeStatic`
- Engineers unfamiliar with Azure SKU constraints may encounter this error and be blocked

**This document captures the full problem-solution-validation cycle**, serving as both an operational runbook and an onboarding reference for engineers working in Azure networking environments.

---

## Architecture Context

| Property | Value |
|---|---|
| Resource Group | `kml_rg_main-6b125c0390d64b55` |
| Region | East US |
| Public IP Name | `devops-pip` |
| Allocation Method | Static |
| SKU | Standard |
| IP Version | IPv4 |
| Assigned IP Address | `104.211.27.1` |
| Provisioning State | Succeeded |

---

## Prerequisites

- Access to an Azure subscription with appropriate permissions
- Azure CLI installed and authenticated (`az login` completed)
- Contributor or Network Contributor role on the target resource group
- Familiarity with basic Azure CLI commands

---

## High-Level Logic

```
CONNECT     to azure-client host
RETRIEVE    Azure credentials using showcreds
LOGIN       to Azure subscription
LIST        existing resource groups
SELECT      active resource group: kml_rg_main-6b125c0390d64b55
ATTEMPT     Public IP creation with Dynamic allocation
IF          allocation fails due to SKU restriction (StandardAndStandardV2SkuPublicIPAddressesMustBeStatic):
    RECREATE    Public IP using Static allocation and Standard SKU
VERIFY      Public IP exists and ipAddress is populated
CONFIRM     provisioningState = Succeeded
```

---

## Implementation

### Step 1: Retrieve Azure Credentials

Before any Azure CLI operations can be performed, valid credentials must be retrieved from the provisioned environment. Run the following command on the `azure-client` host:

```bash
showcreds
```

This command outputs the full credential set required to authenticate against the Azure subscription, including the Console URL, User Name, Application Client ID, and session expiry.

**Key fields captured:**

- **Azure Console URL:** `https://portal.azure.com/azurefreekmlprod.onmicrosoft.com`
- **Azure User Name:** `kk_lab_user_main-6b125c0390d64b55@azurefreekmlprod.onmicrosoft.com`
- **Azure Application Client ID:** `b149cc22-d72e-411a-87c8-859adb512745`
- **Azure Session End Time:** `Sat Feb 14 01:57:01 UTC 2026`

> **Operational Note:** Always confirm session expiry before beginning multi-step operations. Expired sessions will silently fail or produce authentication errors mid-workflow.

**Screenshot: Credential retrieval output**

![showcreds output showing Azure credentials including Console URL, User Name, Client ID, and Session End Time](https://github.com/user-attachments/assets/e6a6e216-86d5-4312-ba6f-94c9413f5f67)

*The `showcreds` command displays all required Azure access credentials, confirming an active session with a valid expiry.*

---

### Step 2: Identify the Active Resource Group

List all resource groups available in the subscription to identify the correct target for resource provisioning:

```bash
az group list --output table
```

**Expected output:**

```
Name                              Location    Status
--------------------------------  ----------  ---------
kml_rg_main-6b125c0390d64b55     eastus      Succeeded
```

Only one resource group is present: **`kml_rg_main-6b125c0390d64b55`** in the `eastus` region with a `Succeeded` provisioning state. This is the target resource group for all subsequent operations.

> **Operational Note:** In multi-subscription environments, always verify you are scoped to the correct subscription using `az account show` before listing resource groups.

**Screenshot: Resource group listing**

![az group list output showing kml_rg_main resource group in eastus with Succeeded status](https://github.com/user-attachments/assets/6bfa2a5b-4911-40b1-8495-14a84f29cf5e)

*The resource group listing confirms a single active group in East US, which will be used as the target for Public IP provisioning.*

---

### Step 3: Attempt Public IP Creation with Dynamic Allocation

The initial provisioning attempt uses Dynamic allocation, which is the default behavior for many Basic SKU scenarios. In Standard SKU environments, however, this will fail.

```bash
az network public-ip create \
  --name devops-pip \
  --resource-group kml_rg_main-6b125c0390d64b55 \
  --allocation-method Dynamic
```

**Result:** Command fails with the following error:

```
(StandardAndStandardV2SkuPublicIPAddressesMustBeStatic) Standard sku publicIp
/subscriptions/.../providers/Microsoft.Network/publicIPAddresses/devops-pip
must have AllocationMethod set to Static.

Code: StandardAndStandardV2SkuPublicIPAddressesMustBeStatic
Message: Standard sku publicIp ... must have AllocationMethod set to Static.
```

**Root Cause Analysis:**

Azure enforces that **Standard SKU Public IP addresses must use Static allocation**. Dynamic allocation is only permitted with Basic SKU. Since the subscription environment defaults to Standard SKU (required for zone-redundancy, Load Balancer compatibility, and enhanced security), the Dynamic flag is rejected at the API level.

> **Key Insight:** This is a breaking change already in effect. The Azure CLI warning about upcoming default behavior for zonal vs. non-zonal regions is informational and does not affect the immediate error. The fix is explicit: use `--allocation-method Static` with `--sku Standard`.

**Screenshot: Dynamic allocation failure**

![az network public-ip create with Dynamic allocation showing StandardAndStandardV2SkuPublicIPAddressesMustBeStatic error](https://github.com/user-attachments/assets/432f7bab-482e-402c-ba37-4318bcfe2a7d)

*The CLI returns a clear error indicating that Standard SKU environments require Static allocation. This is the expected failure that drives the corrective action in Step 4.*

---

### Step 4: Recreate Public IP with Static Allocation and Standard SKU

With the root cause identified, the corrected command explicitly sets both the allocation method and the SKU tier:

```bash
az network public-ip create \
  --name devops-pip \
  --resource-group kml_rg_main-6b125c0390d64b55 \
  --allocation-method Static \
  --sku Standard
```

**Result:** Public IP provisioned successfully. Key properties returned in the JSON response:

| Property | Value |
|---|---|
| `name` | `devops-pip` |
| `ipAddress` | `104.211.27.1` |
| `publicIPAllocationMethod` | `Static` |
| `publicIPAddressVersion` | `IPv4` |
| `provisioningState` | `Succeeded` |
| `sku.name` | `Standard` |
| `sku.tier` | `Regional` |
| `location` | `eastus` |

> **Best Practice:** Always explicitly specify `--sku Standard` and `--allocation-method Static` together in production scripts. Relying on defaults introduces fragility as Azure continues to evolve its default SKU behavior across regions.

**Screenshot: Successful Public IP creation with Static allocation and Standard SKU**

![az network public-ip create with Static allocation and Standard SKU returning Succeeded provisioning state](https://github.com/user-attachments/assets/74256c3c-dbe8-4a92-941c-6e18efcdb916)

*The corrected command succeeds, returning a fully provisioned Public IP with Static allocation, Standard SKU, and an assigned IPv4 address of `104.211.27.1`.*

---

### Step 5: Verify Public IP Provisioning

After creation, the Public IP configuration is independently verified by querying the resource directly:

```bash
az network public-ip show \
  --name devops-pip \
  --resource-group kml_rg_main-6b125c0390d64b55
```

**Verification checklist:**

- **`provisioningState`** = `Succeeded`
- **`ipAddress`** = `104.211.27.1` (populated, confirming Static assignment)
- **`publicIPAllocationMethod`** = `Static`
- **`sku.name`** = `Standard`
- **`publicIPAddressVersion`** = `IPv4`

> **Operational Note:** In automation pipelines, extract the `provisioningState` field programmatically using `--query provisioningState --output tsv` and assert its value before proceeding to dependent resource creation steps.

**Screenshot: Public IP show output confirming full configuration**

![az network public-ip show output confirming devops-pip with Static allocation, Standard SKU, ipAddress 104.211.27.1, and Succeeded provisioning state](https://github.com/user-attachments/assets/d86ce8a9-9f96-4f51-9c17-feb916c3d10b)

*Independent verification confirms all expected properties are correctly set. The Public IP resource is ready for attachment to Azure services such as Load Balancers, Application Gateways, or Virtual Machine NICs.*

---

## Final Outcome

| Attribute | Value |
|---|---|
| Public IP Name | `devops-pip` |
| Allocation Method | Static |
| SKU | Standard |
| IP Version | IPv4 |
| Assigned Address | `104.211.27.1` |
| Provisioning State | **Succeeded** |
| Resource Group | `kml_rg_main-6b125c0390d64b55` |
| Region | East US |

The `devops-pip` Public IP resource is fully provisioned and available for attachment to downstream Azure networking components.

---

## Best Practices and Operational Considerations

**SKU Selection**
- Always use **Standard SKU** for production workloads. Standard SKU supports zone redundancy, is required by Azure Load Balancer Standard, and offers enhanced DDoS protection.
- Basic SKU is being deprecated across Azure services. Do not use it for new deployments.

**Allocation Method**
- **Static allocation** is required for Standard SKU and is strongly preferred in production. It ensures the IP address persists across resource stop/start cycles, enabling consistent DNS records and firewall rules.
- Dynamic allocation releases the IP when the associated resource is deallocated, which can break external connectivity and DNS bindings unexpectedly.

**Scripting and IaC**
- Codify Public IP creation in infrastructure-as-code (Terraform, Bicep, or ARM templates) to ensure repeatability and prevent configuration drift.
- Always pin `--sku` and `--allocation-method` explicitly in scripts, rather than relying on Azure CLI defaults, which are subject to change.

**Naming Conventions**
- Follow a consistent naming convention for Public IPs (e.g., `<service>-pip`, `<env>-<region>-pip`) to aid resource discovery and auditing at scale.

**Tagging**
- Apply resource tags (`environment`, `owner`, `project`, `cost-center`) at creation time to enable cost allocation and governance reporting.

---

## Risks, Edge Cases, and Troubleshooting

**Error: `StandardAndStandardV2SkuPublicIPAddressesMustBeStatic`**
- **Cause:** Dynamic allocation used with Standard SKU
- **Fix:** Add `--allocation-method Static --sku Standard` to the create command

**Error: `AuthorizationFailed`**
- **Cause:** Insufficient RBAC permissions on the resource group or subscription
- **Fix:** Verify the assigned role includes `Microsoft.Network/publicIPAddresses/write` permissions

**IP Address Not Populated After Creation**
- With Dynamic allocation (Basic SKU), the IP is not assigned until the resource is attached to a NIC or Load Balancer. With Static allocation, the IP is assigned immediately at creation, as confirmed in this workflow.

**Zone Redundancy Warning**
- The CLI may emit a warning about upcoming default zone behavior for Standard SKU in zonal regions. This is informational. To suppress uncertainty, explicitly pass `--zone` or `--no-zone` based on your availability requirements.

**Session Expiry During Multi-Step Operations**
- If the Azure session expires mid-workflow (check `Azure Session End Time` from `showcreds`), re-authenticate before continuing. Partial resource creation may leave orphaned resources that incur cost.

---

## Tags

`azure` `networking` `public-ip` `cloud-migration` `devops` `azure-cli` `standard-sku` `static-allocation` `infrastructure`































# Azure Public IP Allocation (devops-pip)

## 📌 Lab Overview
- The Nautilus DevOps team is migrating infrastructure to Azure using
incremental and controlled phases. As part of the networking foundation,
a Public IP address was allocated to support external connectivity
for future Azure resources.

- This lab focuses on allocating a Public IP using Azure CLI
while handling SKU and allocation constraints.

---

## 🎯 Objectives
- Retrieve Azure access credentials
- Identify the active resource group
- Create a Public IP named `devops-pip`
- Resolve allocation method constraints
- Verify successful Public IP provisioning

---

## 🧠 High-Level Logic

- CONNECT to azure-client host
- RETRIEVE Azure credentials using showcreds

- LOGIN to Azure subscription
- LIST existing resource groups
- SELECT active resource group

- ATTEMPT to create Public IP with Dynamic allocation
- IF allocation fails due to SKU restriction:
  -  RECREATE Public IP using Static allocation and Standard SKU

- VERIFY Public IP exists
- CONFIRM provisioning state = Succeeded

## 🛠️ Implementation Steps

## Step 1: Retrieve Azure Credentials
- Run credentials command on the azure-client host:

- showcreds

📸 screenshot:
<img width="1031" height="319" alt="549753232-88c04b3d-296a-4572-a54f-6e8f54ec5d66" src="https://github.com/user-attachments/assets/e6a6e216-86d5-4312-ba6f-94c9413f5f67" />


## Step 2: Identify Resource Group
- List all resource groups in the subscription:

- az group list --output table

📸 screenshot:
<img width="1037" height="573" alt="image" src="https://github.com/user-attachments/assets/6bfa2a5b-4911-40b1-8495-14a84f29cf5e" />

## Step 3: Attempt Public IP Creation (Dynamic Allocation)
- Initial attempt using Dynamic allocation:

- az network public-ip create \
  -  --name devops-pip \
  -  --resource-group kml_rg_main-6b125c0390d64b55 \
  -  --allocation-method Dynamic

- EXPECTED RESULT:

Command fails due to Standard SKU requiring Static allocation

📸 screenshot:
<img width="1026" height="863" alt="549761125-e5f490b7-f71c-40ba-8f68-dcc1e55cdcd0" src="https://github.com/user-attachments/assets/432f7bab-482e-402c-ba37-4318bcfe2a7d" />


## Step 4: Create Public IP with Static Allocation (Corrected)
- Recreate Public IP using Static allocation and Standard SKU:

- az network public-ip create \
  -  --name devops-pip \
  -  --resource-group kml_rg_main-6b125c0390d64b55 \
  -  --allocation-method Static \
  -  --sku Standard

📸 screenshot:
<img width="1031" height="696" alt="image" src="https://github.com/user-attachments/assets/74256c3c-dbe8-4a92-941c-6e18efcdb916" />

## Step 5: Verify Public IP Configuration
- Confirm Public IP details and provisioning state:

- az network public-ip show \
  -  --name devops-pip \
  -  --resource-group kml_rg_main-6b125c0390d64b55

📸 screenshot:
<img width="1030" height="858" alt="image" src="https://github.com/user-attachments/assets/d86ce8a9-9f96-4f51-9c17-feb916c3d10b" />

## ✅ Final Outcome
- Public IP devops-pip successfully created

- Allocation method set to Static

- SKU set to Standard

- IPv4 address assigned

- Provisioning state marked as Succeeded

- Resource ready for attachment to Azure services

## 🏷️ Tags
`azure` `networking` `public-ip` `cloud-migration` `devops` `azure-cli`









