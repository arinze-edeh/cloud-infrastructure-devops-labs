# Azure Networking: Provisioning Virtual Networks via Azure CLI

> Foundational network layer provisioning for cloud-native infrastructure using Azure CLI 2.67.0

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture](#architecture)
- [Objectives](#objectives)
- [Prerequisites](#prerequisites)
- [High-Level Logic](#high-level-logic)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Verify Azure CLI](#step-1-verify-azure-cli)
  - [Step 2: Confirm Azure Login and Subscription Context](#step-2-confirm-azure-login-and-subscription-context)
  - [Step 3: Set Active Subscription and Identify Resource Group](#step-3-set-active-subscription-and-identify-resource-group)
  - [Step 4: Create the Virtual Network](#step-4-create-the-virtual-network)
  - [Step 5: Verify VNet Provisioning](#step-5-verify-vnet-provisioning)
- [Final Outcome](#final-outcome)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Errors and Resolutions](#errors-and-resolutions)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

As part of a phased cloud migration initiative for the Nautilus DevOps team, this engagement establishes the core networking infrastructure on Microsoft Azure. A Virtual Network (VNet) serves as the foundational isolation boundary for all workloads, enabling private communication, subnet segmentation, and integration with downstream services such as virtual machines, load balancers, and private endpoints.

This document captures the end-to-end process of provisioning a production-aligned VNet using the Azure CLI, following a problem-solution-implementation-validation structure suitable for onboarding engineers, infrastructure handoff, and audit traceability.

---

## Problem Statement

Cloud workloads require isolated, addressable network space before any compute or storage resource can be deployed securely. Without a properly defined VNet:

- Virtual machines cannot communicate privately.
- Subnet-level network security policies cannot be applied.
- Private endpoint connectivity to PaaS services is unavailable.
- The infrastructure baseline required for multi-tier applications is absent.

**Solution:** Provision a VNet with a defined IPv4 CIDR block in the target region, scoped to the pre-provisioned resource group, using the Azure CLI for repeatability and auditability.

---

## Architecture

```
Azure Subscription: Azure Free Labs
  └── Resource Group: kml_rg_main-d61c7f18084c43a8 (eastus)
        └── Virtual Network: nautilus-vnet
              └── Address Space: 192.168.0.0/24
                    └── (Subnets to be added in subsequent phases)
```

---

## Objectives

- Authenticate and confirm active Azure subscription context
- Identify the pre-provisioned resource group
- Create a Virtual Network with a defined IPv4 CIDR block
- Deploy the VNet in the correct region (`eastus`)
- Validate successful provisioning via CLI verification commands

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Azure CLI | Version 2.67.0 or later |
| Authentication | Service principal or user account with `Network Contributor` or `Contributor` role |
| Subscription | Active Azure subscription with sufficient quota |
| Resource Group | Pre-existing resource group in target region |
| Shell | Bash-compatible shell (Linux, macOS, or Azure Cloud Shell) |

---

## High-Level Logic

```
AUTHENTICATE to Azure
  VERIFY active subscription and region
  IDENTIFY pre-provisioned resource group
  CREATE virtual network with target CIDR block
  VERIFY VNet exists and reports Succeeded state
```

---

## Implementation Steps

### Step 1: Verify Azure CLI

Before executing any provisioning commands, confirm the Azure CLI version to ensure compatibility with all flags and features used in this engagement.

```bash
az version
```

**Expected Output:**

```json
{
  "azure-cli": "2.67.0",
  "azure-cli-core": "2.67.0",
  "azure-cli-telemetry": "1.1.0",
  "extensions": {}
}
```

> **Operational Note:** Azure CLI 2.67.0 introduced stable support for several networking flags. Running an outdated version may cause silent flag deprecations or unexpected parameter behavior. Always verify before provisioning.

📸 *Azure CLI version confirmed as 2.67.0, confirming toolchain readiness.*

<img width="1033" height="647" alt="az version output confirming Azure CLI 2.67.0" src="https://github.com/user-attachments/assets/7252cdbb-b2d0-426a-85cf-3db13d5684c6" />

---

### Step 2: Confirm Azure Login and Subscription Context

Verify the active Azure account and confirm the correct subscription is loaded. This prevents accidental provisioning against an unintended subscription.

```bash
az account show
```

**Expected Output (key fields):**

```json
{
  "environmentName": "AzureCloud",
  "id": "f0c3bcdd-5ce2-4fa0-8cf3-41559747512b",
  "isDefault": true,
  "name": "Azure Free Labs",
  "state": "Enabled",
  "user": {
    "name": "d8f121f5-e87c-4532-bb69-64f26dea749b",
    "type": "servicePrincipal"
  }
}
```

> **Security Note:** Authentication here is performed via a service principal, which is the recommended pattern for CI/CD pipelines and automated provisioning. Avoid interactive user credentials in scripted environments.

📸 *Account context verified: Azure Free Labs subscription is active and in an Enabled state.*

<img width="1032" height="689" alt="az account show output confirming active subscription" src="https://github.com/user-attachments/assets/c1a716b1-39d4-41aa-b255-9c761d3f51ca" />

---

### Step 3: Set Active Subscription and Identify Resource Group

Explicitly set the target subscription to ensure all subsequent commands are scoped correctly, then list available resource groups to identify the pre-provisioned group.

```bash
az account set --subscription f0c3bcdd-5ce2-4fa0-8cf3-41559747512b
```

```bash
az group list --output table
```

**Expected Output:**

```
Name                            Location    Status
------------------------------  ----------  ---------
kml_rg_main-d61c7f18084c43a8   eastus      Succeeded
```

> **Key Decision:** The resource group `kml_rg_main-d61c7f18084c43a8` is pre-created by the lab environment. Attempting to create a new resource group will fail due to subscription-level policy restrictions. Always audit existing resource groups before provisioning.

> **Operational Consideration:** Explicitly setting the subscription with `az account set` is a defensive practice. It eliminates ambiguity in multi-subscription environments and makes scripts portable across operator sessions.

📸 *Subscription explicitly set, and the pre-provisioned resource group confirmed in eastus.*

<img width="1031" height="737" alt="az account set command executed with no output indicating success" src="https://github.com/user-attachments/assets/38d93dde-6d8a-4b50-86f6-29626959ec50" />

<img width="1028" height="738" alt="az group list output showing kml_rg_main resource group in eastus with Succeeded status" src="https://github.com/user-attachments/assets/ae9f80a3-7081-426f-b8f4-42d3c20ee9b1" />

---

### Step 4: Create the Virtual Network

Provision the Virtual Network with a /24 CIDR block, scoped to the pre-provisioned resource group and deployed in the `eastus` region to match existing infrastructure.

```bash
az network vnet create \
  --resource-group kml_rg_main-d61c7f18084c43a8 \
  --name nautilus-vnet \
  --address-prefix 192.168.0.0/24 \
  --location eastus
```

**Parameter Breakdown:**

| Flag | Value | Purpose |
|---|---|---|
| `--resource-group` | `kml_rg_main-d61c7f18084c43a8` | Target resource group for the VNet |
| `--name` | `nautilus-vnet` | Unique VNet identifier within the resource group |
| `--address-prefix` | `192.168.0.0/24` | IPv4 CIDR defining the private address space (256 addresses) |
| `--location` | `eastus` | Azure region matching the resource group location |

**Expected Output (key fields):**

```json
{
  "newVNet": {
    "addressSpace": {
      "addressPrefixes": ["192.168.0.0/24"]
    },
    "enableDdosProtection": false,
    "location": "eastus",
    "name": "nautilus-vnet",
    "provisioningState": "Succeeded",
    "resourceGroup": "kml_rg_main-d61c7f18084c43a8",
    "subnets": [],
    "virtualNetworkPeerings": []
  }
}
```

> **Design Note:** A /24 block (256 addresses) provides sufficient space for initial workloads while keeping address space controlled. Azure reserves 5 addresses per subnet, so plan CIDR sizing accordingly when subnets are introduced.

> **Risk:** `enableDdosProtection: false` is expected in this context. For production environments handling external traffic, evaluate enabling Azure DDoS Protection Standard at the VNet level.

📸 *VNet creation command submitted; CLI shows Running status indicating async provisioning in progress.*

<img width="1025" height="879" alt="az network vnet create command running with output showing Running status" src="https://github.com/user-attachments/assets/39a626f2-8104-42c3-b0eb-7f5527b44643" />

📸 *VNet provisioning completed successfully; provisioningState confirms Succeeded with full resource metadata returned.*

<img width="1036" height="868" alt="az network vnet create output showing provisioningState Succeeded and full VNet JSON response" src="https://github.com/user-attachments/assets/44ec61f8-70c0-4189-993d-64bee992cbfb" />

---

### Step 5: Verify VNet Provisioning

Run two verification commands: a targeted show against the specific VNet, and a list across the subscription to confirm the resource is visible and correctly registered.

```bash
az network vnet show \
  --resource-group kml_rg_main-d61c7f18084c43a8 \
  --name nautilus-vnet \
  --output table
```

**Expected Output:**

```
EnableDdosProtection    Location    Name            PrivateEndpointVNetPolicies    ProvisioningState    ResourceGroup                     ResourceGuid
----------------------  ----------  --------------  -----------------------------  -------------------  --------------------------------  ------------------------------------
False                   eastus      nautilus-vnet   Disabled                       Succeeded            kml_rg_main-d61c7f18084c43a8      8a866a3d-8df5-442d-a8f4-450e0bb3eb78
```

```bash
az network vnet list --output table
```

**Expected Output:**

```
Name            ResourceGroup                     Location    NumSubnets    Prefixes          DnsServers    DDOSProtection    VMProtection
--------------  --------------------------------  ----------  ------------  ----------------  ------------  ----------------  ------------
nautilus-vnet   kml_rg_main-d61c7f18084c43a8     eastus      0             192.168.0.0/24                  False
```

> **Validation Criteria:** `ProvisioningState: Succeeded`, correct CIDR prefix (`192.168.0.0/24`), and location (`eastus`) all confirm the VNet is correctly provisioned and ready for subnet attachment.

📸 *Both vnet show and vnet list confirm nautilus-vnet is active in eastus with the correct address space and zero subnets pending future phases.*

<img width="1399" height="803" alt="az network vnet show and vnet list output confirming nautilus-vnet provisioned successfully in eastus" src="https://github.com/user-attachments/assets/04b5bfff-7976-4607-b385-9fdb0c49062f" />

---

## Final Outcome

| Attribute | Value |
|---|---|
| VNet Name | `nautilus-vnet` |
| Address Space | `192.168.0.0/24` |
| Location | `eastus` |
| Resource Group | `kml_rg_main-d61c7f18084c43a8` |
| Provisioning State | `Succeeded` |
| Subnets | None (baseline phase; subnets to be added in subsequent phases) |
| DDoS Protection | Disabled (appropriate for this scope) |

The Virtual Network is fully provisioned and serves as the networking foundation for all subsequent Azure resource deployments in this environment.

---

## Key Decisions

- **Pre-created resource group used without modification:** The resource group was pre-provisioned by the environment. Attempting to create a new group would conflict with subscription-level policy restrictions. Always audit existing resource groups before provisioning.

- **/24 CIDR selected over larger blocks:** A 192.168.0.0/24 address space was chosen to provide adequate capacity for initial workloads while keeping the address allocation controlled. Larger CIDR blocks reduce future peering flexibility.

- **No subnets created at VNet provisioning time:** Subnets will be defined in subsequent phases once compute and service requirements are finalized. Creating subnets prematurely can lead to inefficient address space allocation.

- **`--location` explicitly specified:** Even though the resource group is in `eastus`, explicitly passing `--location` avoids ambiguity and ensures the VNet is co-located with its resource group, which is required for resource association.

---

## Best Practices and Operational Considerations

- **Align VNet region with resource group region.** Azure requires VNets and their resource groups to be in the same region for correct resource association and latency optimization.

- **Use consistent naming conventions.** Kebab-case names (`nautilus-vnet`) improve readability in CLI output, ARM templates, and IaC tooling such as Terraform and Bicep.

- **Always validate with a list command after creation.** The `az network vnet list` verification step confirms the resource is visible at the subscription level, not just from the provisioning response.

- **Plan CIDR blocks before provisioning.** Address space cannot overlap with peered VNets or on-premises ranges. Document the IP plan before creating any VNet in a multi-network environment.

- **Avoid DDoS Standard for non-production scopes.** DDoS Protection Standard carries significant cost. Reserve it for production VNets handling internet-facing traffic.

- **Use `--output table` for human-readable audits.** JSON is appropriate for scripting and automation; table format is better for operational verification and documentation screenshots.

---

## Errors and Resolutions

No errors were encountered during this provisioning sequence. The following are common failure modes to anticipate in similar environments:

**Resource group creation blocked by policy**
- **Symptom:** `AuthorizationFailed` or `RequestDisallowedByPolicy` when running `az group create`
- **Resolution:** Use the pre-provisioned resource group provided by the environment. Do not attempt to create new resource groups unless explicitly permitted by the subscription policy.

**VNet name conflict**
- **Symptom:** `VnetAlreadyExistsWithDifferentAddressSpace` error
- **Resolution:** Run `az network vnet list --output table` first to check for existing VNets with the same name before provisioning.

**Location mismatch**
- **Symptom:** Resource deploys to an unexpected region
- **Resolution:** Always pass `--location` explicitly. Never rely on CLI defaults in multi-region environments.

---

## Lessons Learned

- Explicitly setting the subscription with `az account set` before any provisioning command is a defensive discipline that prevents cross-subscription accidents, especially in shared or multi-tenant environments.

- The `az network vnet create` command returns a `newVNet` JSON block on success, which includes the full resource ID and provisioning state. This response is suitable for downstream automation (e.g., passing the VNet ID to a subnet creation command).

- Running both `vnet show` and `vnet list` for verification provides two independent confirmation signals: one scoped to the specific resource, and one at the subscription level. This two-signal pattern is useful for post-deployment audit trails.

---

## Tags

`azure` `vnet` `networking` `cloud-infrastructure` `devops` `azure-cli` `infrastructure-provisioning` `eastus`
