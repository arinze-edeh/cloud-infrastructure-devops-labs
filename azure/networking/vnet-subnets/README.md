# Azure Virtual Network and Subnet Provisioning via CLI

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Objectives](#objectives)
- [Implementation](#implementation)
  - [Step 1: Retrieve Azure Credentials](#step-1-retrieve-azure-credentials)
  - [Step 2: Identify the Existing Resource Group](#step-2-identify-the-existing-resource-group)
  - [Step 3: Create the Virtual Network and Subnet](#step-3-create-the-virtual-network-and-subnet)
- [Validation and Outcome](#validation-and-outcome)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This document covers the provisioning of a foundational Azure networking layer for the Nautilus DevOps team as part of a phased cloud infrastructure migration. Using the Azure CLI, a Virtual Network (VNet) and an initial subnet were created within a pre-existing resource group. This work establishes the network boundary within which future compute, storage, and application workloads will be deployed.

---

## Problem Statement

The Nautilus DevOps team is executing an incremental migration of on-premises infrastructure to Azure. Before any virtual machines, containers, or managed services can be deployed, a well-defined, correctly scoped network must exist. Without this, subsequent workloads would lack isolation, proper IP address management, and a consistent routing boundary.

**Solution:** Provision a Virtual Network with an appropriately sized address space and a dedicated subnet, using the Azure CLI for repeatability and auditability, within an already-provisioned resource group.

---

## Architecture Summary

| Component | Value |
|---|---|
| **Virtual Network** | `nautilus-vnet` |
| **Address Space** | `10.0.0.0/16` |
| **Subnet** | `nautilus-subnet` |
| **Subnet CIDR** | `10.0.1.0/24` |
| **Resource Group** | `kml_rg_main-0e6ddfb2b553417d` |
| **Azure Region** | `southcentralus` |

The `/16` address space provides 65,536 IP addresses across the VNet, while the `/24` subnet carves out 256 addresses for the initial workload tier. This design leaves significant room for additional subnets (e.g., for application, data, or management tiers) as the migration progresses.

---

## Prerequisites

- Azure CLI installed and accessible on the client host
- Valid Azure lab credentials (username, password, application client ID)
- An existing resource group within the target subscription
- Sufficient IAM permissions to create networking resources (Contributor or Network Contributor role)

---

## Objectives

- Retrieve and confirm Azure session credentials
- Identify the pre-provisioned resource group to deploy into
- Create a VNet with a `/16` address space in the correct Azure region
- Simultaneously provision a subnet with a `/24` CIDR block during VNet creation
- Verify successful provisioning through CLI output inspection

---

## Implementation

### Step 1: Retrieve Azure Credentials

Before executing any Azure CLI commands, session credentials were retrieved using the `showcreds` utility available in the KodeKloud lab environment.

```bash
showcreds
```

**Output confirms:**

| Field | Value |
|---|---|
| Azure Console URL | `https://portal.azure.com/azurefreekmlprod.onmicrosoft.com` |
| Azure User Name | `kk_lab_user_main-0e6ddfb2b553417d@azurefreekmlprod.onmicrosoft.com` |
| Azure Application Client ID | `200e14b5-45d1-4af8-92bd-b086c7e3e460` |
| Session End Time | `Fri Feb 13 17:20:24 UTC 2026` |

**Operational note:** Always verify session validity before executing resource provisioning commands. Expired sessions will cause silent authentication failures or misleading error messages downstream.

📸 **Screenshot 1: Azure credentials retrieved via `showcreds`**

<img width="1031" height="551" alt="Azure credentials output from showcreds command" src="https://github.com/user-attachments/assets/a96a79f3-ca3f-4150-a57f-3d9386a97dc2" />

---

### Step 2: Identify the Existing Resource Group

With credentials confirmed, the next step is to enumerate the available resource groups within the subscription. This ensures the VNet is deployed into the correct, pre-existing group rather than accidentally creating a new one or targeting the wrong scope.

```bash
az group list --output table
```

**Output:**

```
Name                              Location    Status
--------------------------------  ----------  ---------
kml_rg_main-0e6ddfb2b553417d     eastus      Succeeded
```

**Key observation:** The resource group is deployed in `eastus`, but the VNet target region is `southcentralus`. This is intentional in lab environments where the resource group serves as a logical container regardless of region. In production, aligning the resource group location with its resources improves latency metadata and reduces operational confusion.

📸 **Screenshot 2: Resource group confirmed via `az group list`**

<img width="990" height="465" alt="image" src="https://github.com/user-attachments/assets/34b4f1f6-2aaa-4a18-bf3f-3e5a57a8feab" />

---

### Step 3: Create the Virtual Network and Subnet

With the resource group confirmed, the VNet and initial subnet were provisioned in a single CLI command. The `az network vnet create` command supports inline subnet creation via the `--subnet-name` and `--subnet-prefix` flags, eliminating the need for a separate `az network vnet subnet create` call and reducing provisioning time.

```bash
az network vnet create \
  --name nautilus-vnet \
  --resource-group kml_rg_main-0e6ddfb2b553417d \
  --location southcentralus \
  --address-prefix 10.0.0.0/16 \
  --subnet-name nautilus-subnet \
  --subnet-prefix 10.0.1.0/24
```

**Parameter rationale:**

| Parameter | Value | Rationale |
|---|---|---|
| `--name` | `nautilus-vnet` | Team-scoped, descriptive name aligned to Nautilus project |
| `--resource-group` | `kml_rg_main-0e6ddfb2b553417d` | Pre-existing group; avoids unauthorized RG creation |
| `--location` | `southcentralus` | Required target region for Nautilus workloads |
| `--address-prefix` | `10.0.0.0/16` | Provides ample IP space for multi-subnet growth |
| `--subnet-name` | `nautilus-subnet` | Initial workload subnet, logically named |
| `--subnet-prefix` | `10.0.1.0/24` | Scoped to 256 addresses; starts at `.1` to leave `.0` available for future use |

**Validated output fields (from CLI JSON response):**

```json
{
  "newVNet": {
    "addressSpace": {
      "addressPrefixes": ["10.0.0.0/16"]
    },
    "enableDdosProtection": false,
    "location": "southcentralus",
    "name": "nautilus-vnet",
    "provisioningState": "Succeeded",
    "resourceGroup": "kml_rg_main-0e6ddfb2b553417d",
    "resourceGuid": "5bd8afb7-7fa8-4950-9343-92c8595818f4",
    "subnets": [
      {
        "addressPrefix": "10.0.1.0/24",
        "name": "nautilus-subnet",
        "provisioningState": "Succeeded",
        "privateEndpointNetworkPolicies": "Disabled",
        "privateLinkServiceNetworkPolicies": "Enabled"
      }
    ],
    "virtualNetworkPeerings": [],
    "type": "Microsoft.Network/virtualNetworks"
  }
}
```

📸 **Screenshot 3: VNet creation command and initial JSON response**

<img width="1033" height="832" alt="az network vnet create command with JSON output showing newVNet and subnet configuration" src="https://github.com/user-attachments/assets/3e4dd377-2ab9-4ea0-a88e-dfffd2aac696" />

📸 **Screenshot 4: Full JSON response confirming subnet and VNet provisioning state as Succeeded**

<img width="1036" height="867" alt="Full JSON response from VNet creation showing provisioningState Succeeded for both VNet and subnet" src="https://github.com/user-attachments/assets/b89647e9-8a49-4510-acd7-7b72cacde9c4" />

---

## Validation and Outcome

The CLI JSON response confirms successful provisioning across all key fields:

| Check | Expected | Observed |
|---|---|---|
| VNet name | `nautilus-vnet` | `nautilus-vnet` |
| Address space | `10.0.0.0/16` | `10.0.0.0/16` |
| Subnet name | `nautilus-subnet` | `nautilus-subnet` |
| Subnet CIDR | `10.0.1.0/24` | `10.0.1.0/24` |
| Region | `southcentralus` | `southcentralus` |
| VNet provisioning state | `Succeeded` | `Succeeded` |
| Subnet provisioning state | `Succeeded` | `Succeeded` |
| VNet peerings | Empty (expected) | `[]` |

**Optional post-deployment verification commands:**

```bash
# Confirm VNet exists and review configuration
az network vnet show \
  --name nautilus-vnet \
  --resource-group kml_rg_main-0e6ddfb2b553417d \
  --output table

# List all subnets within the VNet
az network vnet subnet list \
  --vnet-name nautilus-vnet \
  --resource-group kml_rg_main-0e6ddfb2b553417d \
  --output table
```

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Inline subnet creation during VNet provisioning | Reduces CLI round-trips and ensures atomic resource creation |
| `/16` address space | Supports up to 250 subnets with `/24` sizing, enabling significant horizontal scale |
| `/24` subnet prefix | Standard workload subnet size; provides 251 usable IPs with room to expand |
| `southcentralus` region | Aligns with Nautilus team's target region for compute workloads |
| CLI-based provisioning over portal | Enables scripting, version control, and reproducibility across environments |
| No DDoS Protection enabled | Appropriate for non-production; Standard DDoS Protection should be evaluated for production deployments |

---

## Best Practices and Operational Considerations

- **Address space planning:** Always allocate a VNet address space larger than immediately needed. Expanding a VNet address space after deployment is possible but requires careful overlap checks and can be disruptive if peering is active.
- **Subnet segmentation:** Separate subnets should be used for distinct tiers (application, data, management, gateway). A single flat subnet is acceptable for initial provisioning but should be refactored before scaling.
- **NSG association:** A newly created subnet has no Network Security Group attached by default. For production workloads, always associate an NSG immediately after subnet creation to enforce traffic control.
- **Private endpoint policies:** Note that `privateEndpointNetworkPolicies` is set to `Disabled` by default. If Private Endpoints will be deployed into this subnet, this policy must be explicitly managed.
- **Region alignment:** When possible, align the resource group location with the resources it contains. While Azure allows cross-region resources within a group, misalignment can cause confusion in cost reporting and latency troubleshooting.
- **Naming conventions:** Follow a consistent naming convention (e.g., `<project>-<type>-<env>`) across all network resources. This simplifies querying, filtering, and access control policy authoring at scale.
- **Infrastructure as Code:** For repeatable, auditable deployments, consider encoding this provisioning step into a Bicep template or Terraform configuration after the initial CLI-based workflow is validated.

---

## Lessons Learned

- The `az network vnet create` command supports inline subnet provisioning, which is preferable to running `az network vnet subnet create` as a separate step. This reduces provisioning latency and keeps the resource graph consistent.
- `provisioningState: Succeeded` in the CLI JSON response is the definitive confirmation that a resource is fully deployed and ready. Do not assume success based solely on the absence of an error message.
- Resource group location and resource location can differ in Azure. Always explicitly pass `--location` when creating networking resources to ensure deployment in the intended region.
- In constrained lab environments, always confirm available resource group names via `az group list` before provisioning. Hardcoding group names from documentation without verification is a common source of `ResourceGroupNotFound` errors.

---

## Tags

`azure` `azure-cli` `networking` `virtual-network` `vnet` `subnet` `cloud-infrastructure` `devops` `ip-addressing` `southcentralus` `infrastructure-as-code` `cloud-migration`






















# Azure Virtual Network & Subnet Creation (Azure CLI)

## 📌 Lab Overview

- The Nautilus DevOps team is migrating infrastructure to Azure using
incremental steps. As part of the foundational networking layer,
a Virtual Network (VNet) and subnet were provisioned using the
Azure CLI within an existing resource group.

## 🎯 Objectives

- Authenticate to Azure using provided lab credentials

- Identify existing resource group

- Create a Virtual Network (VNet)

- Create a subnet during VNet provisioning

- Configure IPv4 address space correctly

- Deploy resources in the correct Azure region

## 🧠 High-Level Logic
- CONNECT to Azure CLI host

- RETRIEVE Azure credentials using showcreds
- AUTHENTICATE to Azure session

- LIST available resource groups
- SELECT existing resource group

- CREATE Virtual Network
  -  SET VNet name = nautilus-vnet
  -  SET region = southcentralus
  -  SET address space = 10.0.0.0/16

- CREATE subnet during VNet creation
  -  SET subnet name = nautilus-subnet
  -  SET subnet CIDR = 10.0.1.0/24

## 🛠️ Implementation Steps

## Step 1: Retrieve Azure Credentials
- showcreds

📸 screenshot:
<img width="1031" height="551" alt="549549591-d3519e2b-60e8-48c1-9aff-7270941064d9" src="https://github.com/user-attachments/assets/a96a79f3-ca3f-4150-a57f-3d9386a97dc2" />

## Step 2: List Existing Resource Groups
- az group list --output table

📸 screenshot:
<img width="1056" height="484" alt="image" src="https://github.com/user-attachments/assets/c40da632-59aa-4ee6-80cf-85002879e540" />

## Step 3: Create Virtual Network and Subnet
- az network vnet create \
  -  --name nautilus-vnet \
  -  --resource-group kml_rg_main-0e6ddfb2b553417d \
  -  --location southcentralus \
  -  --address-prefix 10.0.0.0/16 \
  -  --subnet-name nautilus-subnet \
  -  --subnet-prefix 10.0.1.0/24

📸 screenshot:
<img width="1033" height="832" alt="image" src="https://github.com/user-attachments/assets/3e4dd377-2ab9-4ea0-a88e-dfffd2aac696" />
<img width="1036" height="867" alt="image" src="https://github.com/user-attachments/assets/b89647e9-8a49-4510-acd7-7b72cacde9c4" />

## ✅ Final Outcome

- Virtual Network nautilus-vnet successfully created

- IPv4 address space configured as 10.0.0.0/16

- Subnet nautilus-subnet created with 10.0.1.0/24

- Resources deployed in southcentralus

- Infrastructure ready for future Azure workloads

## 🏷️ Tags
`azure` `cli` `networking` `vnet` `subnet` `cloud` `infrastructure` `devops`
