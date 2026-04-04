# [Applying Environment Tags to Azure Virtual Machines via Azure CLI](https://github.com/arinze-edeh/cloud-infrastructure-devops-labs)

> **Project Type:** Azure Compute | Virtual Machine Governance  
> **Domain:** Resource Management, Metadata Tagging, Cost Governance  
> **Environment:** Azure Free Labs | Region: `southcentralus`  
> **VM Target:** `devops-vm` | **Tag Applied:** `Environment=dev`

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Context](#architecture-context)
- [Tools and Prerequisites](#tools-and-prerequisites)
- [Implementation Steps](#implementation-steps)
  - [Step 1: Verify Azure Account Context](#step-1-verify-azure-account-context)
  - [Step 2: List Available Subscriptions](#step-2-list-available-subscriptions)
  - [Step 3: Identify the VM Resource Group](#step-3-identify-the-vm-resource-group)
  - [Step 4: Apply the Environment Tag](#step-4-apply-the-environment-tag)
  - [Step 5: Verify Tag Application](#step-5-verify-tag-application)
- [Validation Checklist](#validation-checklist)
- [Key Decisions](#key-decisions)
- [Best Practices and Operational Considerations](#best-practices-and-operational-considerations)
- [Risks, Edge Cases, and Troubleshooting](#risks-edge-cases-and-troubleshooting)
- [Lessons Learned](#lessons-learned)
- [Final Outcome](#final-outcome)

---

## Overview

This project documents the process of applying a governance metadata tag to an existing Azure Virtual Machine using the **Azure CLI**. Tagging is a foundational practice in cloud resource management, enabling teams to enforce visibility, accountability, and cost allocation at scale.

The operation was executed without any interruption to the running VM, demonstrating a zero-downtime metadata update against an active Azure compute resource.

**Tagging enables:**

- **Resource organization** across subscriptions and resource groups
- **Environment identification** (`dev`, `staging`, `prod`) for deployment pipelines
- **Cost management** through billing filters and chargeback reporting
- **Governance and automation** via Azure Policy enforcement and tag-based alerting

---

## Problem Statement

During an infrastructure audit, the virtual machine `devops-vm` was identified as non-compliant with the organization's tagging policy. The `Environment` metadata tag was absent, making it impossible to correctly attribute the resource to its deployment environment in cost reports or automated governance checks.

**Objective:**

- Confirm the correct subscription and account context
- Programmatically discover the VM's resource group without relying on manual portal navigation
- Apply the `Environment=dev` tag using a non-destructive CLI update
- Validate the tag was persisted correctly by querying the VM's tag metadata

---

## Architecture Context

```
Azure Subscription: Azure Free Labs
  └── Resource Group: KML_RG_MAIN-C849AEA3729A4D94
        └── Virtual Machine: devops-vm
              ├── OS: Ubuntu 22.04 LTS (Canonical, Gen2)
              ├── Size: Standard_B1s
              ├── Region: southcentralus
              ├── Auth: SSH Key (Service Principal context)
              └── Tag Applied: Environment=dev
```

---

## Tools and Prerequisites

| Requirement | Detail |
|---|---|
| **Azure CLI** | Installed and authenticated |
| **Authentication Type** | Service Principal |
| **Subscription** | Azure Free Labs |
| **VM Status** | Pre-existing, running |
| **Permissions Required** | `Microsoft.Compute/virtualMachines/write` on the target resource |

> **Note:** The resource group was pre-created by the lab environment. No resource group creation was required or attempted.

---

## Implementation Steps

---

### Step 1: Verify Azure Account Context

**Intent:** Confirm the CLI session is authenticated against the correct subscription and tenant before performing any write operations. Running commands against the wrong subscription is one of the most common and consequential mistakes in multi-subscription environments.

**Command:**

```bash
az account show
```

**Expected Output:**

- `"name": "Azure Free Labs"` confirms the correct subscription
- `"isDefault": true` confirms this subscription is the active context
- `"state": "Enabled"` confirms the subscription is operational
- `"type": "servicePrincipal"` confirms non-interactive authentication

**Screenshot: `az account show` output confirming active subscription and Service Principal authentication**

![az account show output](https://github.com/user-attachments/assets/79ce7653-1946-4a3d-9b85-b3adbffdc4db)

---

### Step 2: List Available Subscriptions

**Intent:** Cross-reference the subscription list to confirm `Azure Free Labs` is set as the default. In environments with multiple subscriptions, executing commands under an incorrect default subscription can lead to unintended resource modifications.

**Command:**

```bash
az account list --output table
```

**Expected Output:**

| Name | CloudName | SubscriptionId | TenantId | State | IsDefault |
|---|---|---|---|---|---|
| Azure Free Labs | AzureCloud | f0c3bcdd-... | 54c1a2d3-... | Enabled | True |

**Screenshot: `az account list` confirming Azure Free Labs as the active default subscription**

![az account list output](https://github.com/user-attachments/assets/c4144f14-6073-4529-b12a-d49e7fe87403)

---

### Step 3: Identify the VM Resource Group

**Intent:** Dynamically discover the resource group containing `devops-vm` using a JMESPath query, rather than assuming or hardcoding the resource group name. This approach is portable and eliminates human error in environments where resource groups follow auto-generated naming conventions.

**Command:**

```bash
az vm list --query "[?name=='devops-vm'].{ResourceGroup:resourceGroup}"
```

**Expected Output:**

```json
[
  {
    "ResourceGroup": "KML_RG_MAIN-C849AEA3729A4D94"
  }
]
```

**Screenshot: JMESPath query returning the resource group for `devops-vm`**

![VM resource group query output](https://github.com/user-attachments/assets/be960db0-9bf1-4250-9ce0-af28fb4e0fb1)

> **Operational Note:** The resource group name `KML_RG_MAIN-C849AEA3729A4D94` uses an auto-generated identifier appended by the lab platform. Always query programmatically rather than assuming static names in managed lab or enterprise-provisioned environments.

---

### Step 4: Apply the Environment Tag

**Intent:** Execute a non-destructive tag update against the VM using `az vm update --set`. This operation modifies only the VM's metadata and does not restart, deallocate, or modify the compute configuration in any way.

**Command:**

```bash
az vm update \
  --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
  --name devops-vm \
  --set tags.Environment=dev
```

**Expected Behavior:**

- The Azure Resource Manager API applies the tag to the VM object
- The full VM JSON is returned, including the updated `tags` block
- `"provisioningState": "Succeeded"` confirms the write completed without error
- The VM remains running with no interruption

**Key fields to verify in the returned JSON:**

- `"provisioningState": "Succeeded"` - confirms successful API write
- `"tags": { "Environment": "dev" }` - confirms tag persistence
- `"name": "devops-vm"` - confirms operation targeted the correct VM
- `"location": "southcentralus"` - confirms correct region

**Screenshots: Full `az vm update` JSON response showing the tag applied and `provisioningState: Succeeded`**

![az vm update response part 1 - hardware profile and VM identity](https://github.com/user-attachments/assets/f27a2e76-65fa-46bd-b9b2-dd2199327d5b)

![az vm update response part 2 - OS profile and network configuration](https://github.com/user-attachments/assets/110218dc-3a5e-4ff2-a9de-674851beb824)

![az vm update response part 3 - provisioningState Succeeded and security profile](https://github.com/user-attachments/assets/bfb9b7c6-4480-41a0-8a21-7f5ec658cdda)

![az vm update response part 4 - storage profile and tags block showing Environment=dev](https://github.com/user-attachments/assets/364409ed-51ac-4d51-a744-458ba3e81428)

> **Key observation from screenshot 4:** The `"tags"` block in the full VM JSON confirms `"Environment": "dev"` was written successfully to the VM object prior to running the explicit verification step.

---

### Step 5: Verify Tag Application

**Intent:** Issue a targeted `az vm show` command with a `--query tags` filter to independently verify the tag was persisted correctly. This step serves as the formal validation gate, confirming the write operation succeeded and is queryable through the ARM API.

**Command:**

```bash
az vm show \
  --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
  --name devops-vm \
  --query tags
```

**Expected Output:**

```json
{
  "Environment": "dev"
}
```

**Screenshot: Targeted tag query confirming `Environment=dev` is persisted on `devops-vm`**

![VM tags verification output](https://github.com/user-attachments/assets/776cd497-40aa-4a77-aeda-d54ff0728790)

---

## Validation Checklist

- Correct subscription confirmed active (`Azure Free Labs`, `IsDefault: True`)
- Service Principal authentication verified (`az account show`)
- Resource group identified programmatically via JMESPath query
- Tag key `Environment` applied with correct casing
- Tag value `dev` applied with correct casing
- `provisioningState: Succeeded` confirmed in update response
- Tag independently verified via `az vm show --query tags`
- Zero downtime introduced to the running VM

---

## Key Decisions

| Decision | Rationale |
|---|---|
| **Used JMESPath query to find resource group** | Avoids hardcoding resource group names in managed environments with auto-generated identifiers |
| **Used `az vm update --set` over `az tag update`** | `az vm update --set tags.*` is idempotent and scoped to the VM resource; `az tag update` requires the full resource ID and is better suited for bulk or cross-resource tagging |
| **Ran `az vm show --query tags` as a separate verification step** | Provides an independent read confirmation against the ARM API, validating the tag is persisted and queryable, not just returned in the write response |
| **Did not use `--no-wait`** | Synchronous execution ensures the CLI session confirms `provisioningState: Succeeded` before exiting, which is appropriate for auditable governance operations |

---

## Best Practices and Operational Considerations

- **Define a tag taxonomy early.** Establish organization-wide tag keys (`Environment`, `Owner`, `CostCenter`, `Project`) and enforce them via Azure Policy with `deny` or `append` effects to prevent untagged resources from being created.

- **Use Azure Policy to enforce tag compliance at scale.** Relying on manual CLI tagging is not sustainable across large subscriptions. Policy-driven enforcement ensures tags are applied consistently at provisioning time.

- **Tag at the resource group level where appropriate.** Tags do not inherit from resource groups to child resources by default in Azure. If consistent tagging across all resources in a group is required, use Azure Policy to propagate tags automatically.

- **Use lowercase, consistent tag values.** Tag values are case-sensitive in some Azure services and reporting tools. Adopting a lowercase convention (`dev`, `prod`, `staging`) prevents duplicate entries in cost reports.

- **Audit tag compliance regularly.** Use `az resource list --tag Environment=dev` or Azure Cost Management to generate reports of tagged and untagged resources.

---

## Risks, Edge Cases, and Troubleshooting

| Scenario | Risk | Resolution |
|---|---|---|
| **Multiple VMs share the same name across subscriptions** | `az vm list` returns results from all subscriptions if `--subscription` is not specified | Always scope `az vm list` with `--subscription` in multi-subscription environments |
| **Insufficient permissions** | `az vm update` fails with `AuthorizationFailed` if the Service Principal lacks `Contributor` or a custom role with `Microsoft.Compute/virtualMachines/write` | Verify role assignment: `az role assignment list --assignee <sp-object-id>` |
| **Tag overwrites an existing value** | `--set tags.Environment=dev` will silently overwrite an existing `Environment` tag if one is present | Query existing tags first with `az vm show --query tags` before applying updates |
| **Resource group name case sensitivity** | Azure resource group names are case-insensitive at the API level, but CLI output may reflect the original casing | Use the exact casing returned from `az vm list` to avoid confusion in scripts |
| **Tag propagation lag in Cost Management** | Newly applied tags may take up to 24 hours to appear in Azure Cost Management and billing reports | This is an expected platform behavior and does not indicate a failed tag operation |

---

## Lessons Learned

- **JMESPath queries are a force multiplier for CLI workflows.** Using `--query "[?name=='devops-vm'].{ResourceGroup:resourceGroup}"` eliminates the need to scroll through unfiltered `az vm list` output and makes scripts portable across environments.

- **Separation of write and verify steps is essential for auditable operations.** Running `az vm show --query tags` as a dedicated step after `az vm update` provides a clean, independent confirmation that can be logged and referenced separately from the update operation itself.

- **Service Principal authentication is the correct approach for automated and lab environments.** Non-interactive authentication removes the risk of session expiry mid-operation and reflects production pipeline behavior accurately.

- **`az vm update --set tags.*` does not trigger a VM restart.** Tag metadata updates are ARM-level operations and do not affect the compute layer. This makes them safe to execute against production VMs without scheduling a maintenance window.

---

## Final Outcome

The virtual machine `devops-vm` was successfully tagged using the Azure CLI with zero downtime and full verification against the Azure Resource Manager API.

| Attribute | Value |
|---|---|
| **VM Name** | `devops-vm` |
| **Resource Group** | `KML_RG_MAIN-C849AEA3729A4D94` |
| **Region** | `southcentralus` |
| **Tag Applied** | `Environment=dev` |
| **Provisioning State** | `Succeeded` |
| **VM Downtime** | None |

This operation establishes a repeatable, auditable pattern for applying governance metadata to Azure compute resources at scale, directly applicable to production tagging workflows, policy remediation tasks, and infrastructure-as-code onboarding pipelines.

































# Azure VM Tagging Using Azure CLI

PROJECT TYPE:
    `Azure Compute – Virtual Machine Management`

TASK:
    `Apply an Environment tag to an existing Azure Virtual Machine using Azure CLI`

VM NAME:
    `devops-vm`

TAG APPLIED:
    `Environment=dev`

SUBSCRIPTION:
    `Azure Free Labs`

REGION:
    `southcentralus`

---

## Project Overview

This project demonstrates how to use `Azure CLI` to apply metadata tags to an existing Virtual Machine.

Tagging helps with:
- Resource organization
- Environment identification
- Cost management
- Governance and automation
---

## Problem Statement

During infrastructure migration, a virtual machine was discovered to be missing the required environment tag.

The objective was to:
- Identify the VM
- Apply the correct tag
- Verify successful tagging using CLI

---

## Tools Used

- Azure CLI
- Azure Cloud Lab Environment
- Service Principal Authentication

---

## Prerequisites

- Azure CLI installed and configured
- Active Azure subscription
- Permissions to update virtual machines
- VM already exists

---

## Implementation Steps

---

### Step 1: Verify Azure Account Context

ACTION:
- Confirm active Azure subscription and login context

COMMAND:
    `az account show`

EXPECTED OUTPUT:
- Subscription name
- Subscription ID
- Tenant ID
- Account state = Enabled

SCREENSHOT: `az account show output`
<img width="1027" height="849" alt="image" src="https://github.com/user-attachments/assets/79ce7653-1946-4a3d-9b85-b3adbffdc4db" />

---

### Step 2: List Available Subscriptions

ACTION:
- Verify correct subscription is set as default

COMMAND:
- `az account list --output table`

EXPECTED RESULT:
- Azure Free Labs subscription
- IsDefault = `True`

SCREENSHOT: `az account list output`
<img width="1029" height="847" alt="image" src="https://github.com/user-attachments/assets/c4144f14-6073-4529-b12a-d49e7fe87403" />

---

### Step 3: Identify Resource Group of the VM

ACTION:
- Query Azure to find the resource group containing the target VM

COMMAND:
- `az vm list --query "[?name=='devops-vm'].{ResourceGroup:resourceGroup}"`

EXPECTED RESULT:
- ResourceGroup = `KML_RG_MAIN-C849AEA3729A4D94`

SCREENSHOT: `VM resource group query output`
<img width="1027" height="790" alt="image" src="https://github.com/user-attachments/assets/be960db0-9bf1-4250-9ce0-af28fb4e0fb1" />

---

### Step 4: Apply Environment Tag to the VM

ACTION:
- Update the VM metadata to include Environment tag

COMMAND:
    az vm update \
      --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
      --name devops-vm \
      --set tags.Environment=dev

EXPECTED RESULT:
    
- ProvisioningState = Succeeded
    
- Tag added successfully
    
- VM remains running

SCREENSHOTS: `az vm update success output`
<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/f27a2e76-65fa-46bd-b9b2-dd2199327d5b" />
<img width="1017" height="860" alt="image" src="https://github.com/user-attachments/assets/110218dc-3a5e-4ff2-a9de-674851beb824" />
<img width="1028" height="862" alt="image" src="https://github.com/user-attachments/assets/bfb9b7c6-4480-41a0-8a21-7f5ec658cdda" />
<img width="1028" height="849" alt="image" src="https://github.com/user-attachments/assets/364409ed-51ac-4d51-a744-458ba3e81428" />

---

### Step 5: Verify Tag Application

ACTION:
- Retrieve VM tags to confirm update

COMMAND:
    az vm show \
      --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
      --name devops-vm \
      --query tags

EXPECTED OUTPUT:
    {
      "Environment": "dev"
    }

SCREENSHOT: `VM tags verification output`
<img width="1036" height="446" alt="image" src="https://github.com/user-attachments/assets/776cd497-40aa-4a77-aeda-d54ff0728790" />

---

## Validation Checklist

- Correct subscription selected
- Correct resource group identified
- Tag key applied correctly
- Tag value applied correctly
- VM provisioning state succeeded
- No downtime introduced

---

## Final Outcome

- The virtual machine `"devops-vm"` was successfully tagged using Azure CLI.

- Tag Applied:
    `Environment=dev`

- This ensures improved governance, resource visibility, and alignment with DevOps best practices.














