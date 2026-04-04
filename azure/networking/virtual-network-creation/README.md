# Provisioning an Azure Virtual Network (VNet) via Azure CLI

> **Enterprise-style Cloud Networking | Infrastructure as Code | Azure CLI Automation**

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Prerequisites](#prerequisites)
- [Architecture Summary](#architecture-summary)
- [High-Level Workflow](#high-level-workflow)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Retrieve Azure Credentials](#step-1-retrieve-azure-credentials)
  - [Step 2: Verify Azure Login and Active Subscription](#step-2-verify-azure-login-and-active-subscription)
  - [Step 3: Identify the Target Resource Group](#step-3-identify-the-target-resource-group)
  - [Step 4: Create the Virtual Network](#step-4-create-the-virtual-network)
  - [Step 5: Verify Virtual Network Provisioning](#step-5-verify-virtual-network-provisioning)
- [Final Result](#final-result)
- [Operational Considerations](#operational-considerations)
- [Troubleshooting](#troubleshooting)
- [Tags](#tags)

---

## Overview

This document details the end-to-end process for provisioning an Azure Virtual Network (VNet) using the Azure CLI as part of a phased cloud migration initiative. The VNet is deployed in the **Central US** region with an IPv4 CIDR block, forming the foundational network layer for workloads being migrated from on-premises infrastructure.

All steps are executed exclusively via the Azure CLI, ensuring repeatability, auditability, and alignment with infrastructure-as-code principles.

---

## Problem Statement

During a phased cloud migration, network isolation and segmentation must be established before any compute or data resources are deployed. Without a properly configured VNet, workloads cannot be securely placed within a defined network boundary, and connectivity requirements such as subnet segmentation, private endpoints, and peering cannot be fulfilled.

**Solution:** Programmatically provision an Azure Virtual Network with a defined address space using the Azure CLI, enabling downstream network and compute resources to be attached in subsequent migration phases.

---

## Prerequisites

- Azure CLI installed and authenticated (`az login` or service principal login already completed)
- Sufficient permissions: **Contributor** or **Network Contributor** role on the target subscription or resource group
- An existing Azure subscription with an available resource group
- Basic familiarity with CIDR notation and Azure networking constructs

---

## Architecture Summary

| Component | Value |
|---|---|
| **VNet Name** | `xfusion-vnet` |
| **Address Space** | `10.0.0.0/16` |
| **Region** | `centralus` |
| **Resource Group** | `kml_rg_main-2de2f556a3f04cf0` |
| **Provisioning Method** | Azure CLI |
| **DDoS Protection** | Disabled (Basic plan) |

---

## High-Level Workflow

```
AUTHENTICATE to Azure
        |
VERIFY active subscription and service principal
        |
IDENTIFY existing resource group
        |
        +-- IF resource group does not exist --> CREATE resource group in centralus
        |
CREATE Virtual Network with /16 IPv4 CIDR
        |
VERIFY VNet provisioning state = Succeeded
```

---

## Implementation Steps

### Step 1: Retrieve Azure Credentials

Before interacting with Azure resources, retrieve the session credentials provisioned for this environment. The `showcreds` command outputs the Azure Console URL, username, application client ID, and session expiry.

```bash
showcreds
```

**What to verify:**
- `Azure User Name` is populated and matches the expected service principal format
- `Azure Application Client ID` is present (used for service principal authentication)
- `Azure Session End Time` confirms the session has not expired

**Screenshot: Credential output from `showcreds`**

![Credentials retrieved via showcreds command](https://github.com/user-attachments/assets/f5fd2742-f96f-42cd-8fb0-397fdc26e930)

*The `showcreds` output confirms the active Azure session, user identity, application client ID, and session validity window.*

---

### Step 2: Verify Azure Login and Active Subscription

Confirm that the Azure CLI is authenticated and operating against the correct subscription and tenant. This step prevents resource provisioning against the wrong environment, a common cause of misdeployed infrastructure in multi-tenant or multi-subscription organizations.

```bash
az account show
```

**Expected output fields:**
- `"name": "Azure Free Labs"` confirms the correct subscription context
- `"state": "Enabled"` confirms the subscription is active
- `"type": "servicePrincipal"` confirms the session is using a service principal identity, consistent with CI/CD and automation workflows
- `"tenantId"` and `"homeTenantId"` should match, confirming no cross-tenant delegation is in effect

**Screenshot: Active subscription details from `az account show`**

![az account show output confirming authenticated subscription](https://github.com/user-attachments/assets/8c21f6a8-2a63-41ff-9920-8677ad27b3f8)

*Output confirms authentication as a service principal under the `Azure Free Labs` subscription in `Enabled` state.*

---

### Step 3: Identify the Target Resource Group

List all available resource groups to confirm the target resource group exists and is in the correct region before creating any network resources. In production environments, VNets must always be provisioned within a resource group that reflects the correct cost center, environment tag, and region policy.

```bash
az group list --output table
```

**Expected output:**

```
Name                              Location    Status
--------------------------------  ----------  ---------
kml_rg_main-2de2f556a3f04cf0     eastus      Succeeded
```

> **Note:** The resource group `kml_rg_main-2de2f556a3f04cf0` is located in `eastus`. The VNet will be provisioned in `centralus`. Azure supports resource groups and resources existing in different regions, as the resource group is a logical container and does not enforce geographic colocation.

**If the required resource group does not exist**, create it before proceeding:

```bash
az group create \
  --name xfusion-rg \
  --location centralus
```

**Screenshot: Resource group listing confirming target group exists**

![az group list output showing available resource groups](https://github.com/user-attachments/assets/5dc71056-a1e5-4a5f-b45d-082e0807d91f)

*The `kml_rg_main-2de2f556a3f04cf0` resource group is confirmed as active and in `Succeeded` state, ready to host the VNet.*

---

### Step 4: Create the Virtual Network

Provision the Virtual Network with the specified name, region, address space, and resource group. The `/16` CIDR block (`10.0.0.0/16`) provides 65,536 IP addresses, sufficient to support multiple subnets and workload tiers as the migration progresses.

```bash
az network vnet create \
  --resource-group kml_rg_main-2de2f556a3f04cf0 \
  --name xfusion-vnet \
  --location centralus \
  --address-prefixes 10.0.0.0/16
```

**Parameter breakdown:**

| Parameter | Value | Purpose |
|---|---|---|
| `--resource-group` | `kml_rg_main-2de2f556a3f04cf0` | Target resource group for logical grouping and billing |
| `--name` | `xfusion-vnet` | Unique VNet identifier within the resource group |
| `--location` | `centralus` | Azure region for the VNet deployment |
| `--address-prefixes` | `10.0.0.0/16` | IPv4 CIDR block defining the address space |

**Key output fields to review:**
- `"provisioningState": "Succeeded"` confirms successful deployment
- `"location": "centralus"` confirms correct region placement
- `"addressPrefixes": ["10.0.0.0/16"]` confirms the address space is applied as intended
- `"subnets": []` indicates the VNet is created without subnets; subnets should be added in subsequent steps per architecture design

**Screenshot: VNet creation output confirming successful provisioning**

![az network vnet create output showing provisioning details](https://github.com/user-attachments/assets/918a2d0f-0529-46e2-92cf-22a7717aea28)

*The CLI response confirms `xfusion-vnet` was created in `centralus` with address prefix `10.0.0.0/16` and a provisioning state of `Succeeded`.*

---

### Step 5: Verify Virtual Network Provisioning

After creation, independently verify the VNet state by querying the resource directly. This validation step ensures the VNet is active and accurately reflects the intended configuration, and serves as the final gate before downstream resources (subnets, NSGs, VMs) are attached.

```bash
az network vnet show \
  --resource-group kml_rg_main-2de2f556a3f04cf0 \
  --name xfusion-vnet \
  --output table
```

**Expected table output columns:**

| Field | Expected Value |
|---|---|
| `EnableDdosProtection` | `False` |
| `Location` | `centralus` |
| `Name` | `xfusion-vnet` |
| `PrivateEndpointVNetPolicies` | `Disabled` |
| `ProvisioningState` | `Succeeded` |
| `ResourceGroup` | `kml_rg_main-2de2f556a3f04cf0` |

**Screenshot: VNet verification confirming active and correctly configured state**

![az network vnet show output confirming VNet configuration](https://github.com/user-attachments/assets/bfe2c57c-16de-402e-b268-29f745fe5b1c)

*The `az network vnet show` output confirms that `xfusion-vnet` is fully provisioned, active in `centralus`, and associated with the correct resource group and resource GUID.*

---

## Final Result

| Attribute | Value |
|---|---|
| **VNet Name** | `xfusion-vnet` |
| **Provisioning State** | `Succeeded` |
| **Address Space** | `10.0.0.0/16` |
| **Region** | `centralus` |
| **Resource Group** | `kml_rg_main-2de2f556a3f04cf0` |
| **Deployment Method** | Azure CLI only |
| **DDoS Protection** | Disabled |

The Virtual Network `xfusion-vnet` has been successfully provisioned in the Central US region with a `/16` IPv4 address space, establishing the network foundation required for subsequent cloud migration phases.

---

## Operational Considerations

**Address Space Planning**
- A `/16` block is broad by design and appropriate for an initial migration VNet. For production environments, plan subnet segmentation (e.g., `/24` per tier: web, app, data, management) before attaching workloads.
- Avoid overlapping CIDR ranges with on-premises networks if VNet peering or VPN Gateway connectivity is planned.

**Resource Group and Region Alignment**
- The VNet is deployed to `centralus` while the resource group resides in `eastus`. This is valid in Azure but ensure all dependent resources (VMs, NICs, NSGs) are deployed to `centralus` to avoid cross-region latency and egress costs.

**DDoS Protection**
- Basic DDoS protection is enabled by default at no additional cost. For production workloads exposed to the internet, evaluate **Azure DDoS Network Protection** (standard tier) for enhanced mitigation capabilities.

**Tagging Strategy**
- Apply resource tags (`environment`, `owner`, `cost-center`, `project`) at VNet creation for governance and cost allocation. Example:
  ```bash
  --tags environment=production owner=platform-team cost-center=12345
  ```

**Network Security**
- Subnets should be created with associated **Network Security Groups (NSGs)** before compute resources are attached. A VNet without subnet-level NSGs provides no traffic filtering.

**Service Principal Scope**
- The service principal used in this workflow has subscription-level access. In production, apply the **principle of least privilege** by scoping the service principal to the specific resource group using role assignments.

---

## Troubleshooting

| Symptom | Likely Cause | Resolution |
|---|---|---|
| `AuthorizationFailed` on VNet create | Insufficient RBAC role | Assign `Network Contributor` or `Contributor` to the service principal on the resource group |
| `LocationNotAvailableForResourceType` | VNet not supported in selected region | Verify the region supports `Microsoft.Network/virtualNetworks` via `az provider show` |
| `AddressSpaceOverlap` error | CIDR conflicts with existing VNet in the subscription | Choose a non-overlapping address prefix |
| `provisioningState: Failed` in output | Transient Azure platform issue | Retry the command; check Azure Service Health for regional incidents |
| `ResourceGroupNotFound` | Target resource group deleted or incorrect name | Re-run `az group list` and confirm the exact resource group name |

---

## Tags

`azure` `networking` `vnet` `azure-cli` `cloud-migration` `infrastructure-as-code` `devops` `platform-engineering`





























# Azure Virtual Network Creation (VNet)

## 📌 Lab Overview
- This lab demonstrates how to create an Azure Virtual Network (VNet)
using the Azure CLI as part of a phased cloud migration strategy.

- The VNet is created in the Central US region using an IPv4 CIDR block.

---

## 🎯 Objective
- Authenticate to Azure using Azure CLI
- Create a resource group (if required)
- Create a Virtual Network named `xfusion-vnet`
- Verify successful creation

---

## 🧠 High-Level Flow

- AUTHENTICATE to Azure
- VERIFY active subscription

- CHECK for existing resource group
- IF resource group does not exist
  -  CREATE resource group in centralus
- END IF

- CREATE Virtual Network with IPv4 CIDR
- VERIFY VNet exists and is active

## 🛠️ Implementation Steps
## Step 1: Retrieve Azure Credentials
- `showcreds`

📸 screenshot:
<img width="1022" height="742" alt="548048236-82eb4020-f4c5-493e-9842-c7954aa9701b" src="https://github.com/user-attachments/assets/f5fd2742-f96f-42cd-8fb0-397fdc26e930" />

## Step 2: Verify Azure Login
- `az account show`

📸 screenshot:
<img width="1030" height="615" alt="image" src="https://github.com/user-attachments/assets/8c21f6a8-2a63-41ff-9920-8677ad27b3f8" />

## Step 3: Check or Create Resource Group
- `az group list --output table
az group create \
  --name xfusion-rg \
  --location centralus`
  
📸 screenshot:
<img width="1027" height="619" alt="image" src="https://github.com/user-attachments/assets/5dc71056-a1e5-4a5f-b45d-082e0807d91f" />

## Step 4: Create Virtual Network
- `az network vnet create \
  --resource-group kml_rg_main-2de2f556a3f04cf0 \
  --name xfusion-vnet \
  --location centralus \
  --address-prefixes 10.0.0.0/16`

📸 screenshot:
<img width="1034" height="864" alt="image" src="https://github.com/user-attachments/assets/918a2d0f-0529-46e2-92cf-22a7717aea28" />

## Step 5: Verify Virtual Network
- `az network vnet show \
  --resource-group kml_rg_main-2de2f556a3f04cf0 \
  --name xfusion-vnet \
  --output table`

📸 screenshot:
<img width="1392" height="857" alt="image" src="https://github.com/user-attachments/assets/bfe2c57c-16de-402e-b268-29f745fe5b1c" />

## ✅ Final Result

- Virtual Network xfusion-vnet successfully created

- IPv4 address space assigned

- Resource deployed in centralus region

- Task completed using Azure CLI only

## 🏷️ Tags
`azure` `networking` `vnet` `azure-cli` `cloud-migration`



