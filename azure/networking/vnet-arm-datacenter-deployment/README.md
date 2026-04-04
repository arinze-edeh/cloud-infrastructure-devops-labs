# Azure Virtual Network Provisioning via ARM Template – Datacenter Infrastructure Standard

> **Domain:** Azure Infrastructure as Code | **Service:** Azure Virtual Network | **Tool:** Azure CLI + ARM Templates | **Scope:** Network Layer Provisioning

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Implementation](#implementation)
  - [Step 1: Verify Azure Account Context](#step-1-verify-azure-account-context)
  - [Step 2: Navigate to ARM Templates Directory](#step-2-navigate-to-arm-templates-directory)
  - [Step 3: Confirm ARM Template File Exists](#step-3-confirm-arm-template-file-exists)
  - [Step 4: Identify Target Resource Group](#step-4-identify-target-resource-group)
  - [Step 5: Update Virtual Network Name and displayName Tag](#step-5-update-virtual-network-name-and-displayname-tag)
  - [Step 6: Update Address Space](#step-6-update-address-space)
  - [Step 7: Add Environment Tag](#step-7-add-environment-tag)
  - [Step 8: Validate and Fix JSON Formatting](#step-8-validate-and-fix-json-formatting)
  - [Step 9: Deploy ARM Template](#step-9-deploy-arm-template)
  - [Step 10: Verify Virtual Network Deployment](#step-10-verify-virtual-network-deployment)
- [Validation Results](#validation-results)
- [Key Decisions](#key-decisions)
- [Errors and Resolutions](#errors-and-resolutions)
- [Best Practices](#best-practices)
- [Lessons Learned](#lessons-learned)

---

## Overview

This document covers the end-to-end modification and deployment of an Azure Virtual Network (VNet) using an ARM (Azure Resource Manager) template via the Azure CLI. The workflow targets infrastructure standardization for a datacenter environment, encompassing naming convention enforcement, CIDR address space reconfiguration, environment-level tagging, and declarative deployment through a version-controlled ARM template.

This pattern is repeatable, auditable, and suitable for integration into CI/CD pipelines or automated infrastructure provisioning workflows.

---

## Problem Statement

An existing ARM template for a Virtual Network required updates to meet datacenter infrastructure standards before deployment. Specifically, the template lacked:

- A standardized resource name aligned to datacenter naming conventions
- A correct CIDR address space for the datacenter network segment (`192.168.0.0/16`)
- Environment-level metadata tagging required for resource governance and cost allocation
- A validated JSON structure prior to deployment

Manual portal-based provisioning was not acceptable given the need for repeatability, version control, and audit trail compliance.

---

## Architecture Summary

| Component | Value |
|---|---|
| Resource Type | `Microsoft.Network/virtualNetworks` |
| VNet Name | `arm-vnet-datacenter` |
| Address Space | `192.168.0.0/16` |
| Resource Group | `kml_rg_main-7dd447df1f3c4b74` |
| Region | East US |
| Deployment Mode | Incremental |
| Tag: displayName | `arm-vnet-datacenter` |
| Tag: Environment | `KKE-datacenter` |
| Provisioning State | Succeeded |

---

## Prerequisites

- Azure CLI installed and authenticated on the control machine
- An active Azure session authenticated as a Service Principal or user with `Network Contributor` or `Contributor` role on the target resource group
- Access to the ARM template file `vnet-deployment-template.json` in `/root/arm-templates/`
- An existing Azure Resource Group to deploy into
- `sed` and `vi` available in the shell environment for template modification

---

## Implementation

### Step 1: Verify Azure Account Context

Before making any changes or deployments, confirm the active Azure session is authenticated to the correct subscription and tenant. This prevents accidental deployments to unintended environments.

```bash
az account show
```

**Expected output fields to verify:**
- `environmentName`: Should be `AzureCloud`
- `name`: Subscription name (e.g., `Azure Free Labs`)
- `state`: Must be `Enabled`
- `user.type`: `servicePrincipal` (for automated/CI contexts) or `user`

> **Operational note:** In multi-subscription environments, explicitly set the target subscription before proceeding: `az account set --subscription <subscription-id>`

**Screenshot: Active Azure session context confirmed**

![Step 1 - az account show](https://github.com/user-attachments/assets/910fba49-f5ae-48a1-a16d-c8dcd7f39965)

---

### Step 2: Navigate to ARM Templates Directory

Change to the directory containing the ARM template files. Keeping templates in a dedicated directory supports organized, version-controlled infrastructure management.

```bash
cd /root/arm-templates
```

**Screenshot: Directory navigation to `/root/arm-templates`**

![Step 2 - Navigate to arm-templates directory](https://github.com/user-attachments/assets/4698b010-f4d0-4bb5-9df2-589054894d96)

---

### Step 3: Confirm ARM Template File Exists

Verify the target ARM template file is present and accessible before proceeding with modifications. Check file permissions and size to confirm it is a valid, non-empty file.

```bash
ls -l vnet-deployment-template.json
```

**Expected output:** A file entry showing ownership as `root`, a size of `740` bytes, and a timestamp reflecting last modification.

> **Risk:** Proceeding without this check risks modifying or targeting an incorrect or absent file, leading to a failed deployment.

**Screenshot: ARM template file confirmed present with correct metadata**

![Step 3 - Confirm template file exists](https://github.com/user-attachments/assets/d0779c4b-224e-41c7-8051-1b54b7e0e494)

---

### Step 4: Identify Target Resource Group

Query all available resource groups and filter for the target datacenter resource group. This confirms the deployment target before any changes are applied.

```bash
az group list --query '[].name' --output table | grep 'kml'
```

**Expected output:** `kml_rg_main-7dd447df1f3c4b74`

> **Operational note:** In environments with many resource groups, using `grep` or a more specific JMESPath query prevents deploying to an incorrect group. Always confirm the exact resource group name before use in deployment commands.

**Screenshot: Target resource group `kml_rg_main-7dd447df1f3c4b74` identified**

![Step 4 - Identify target resource group](https://github.com/user-attachments/assets/53dd6bad-82ce-42f7-810e-8ea459223d08)

---

### Step 5: Update Virtual Network Name and displayName Tag

Use `sed` to perform in-place substitutions on the ARM template to set the standardized VNet name and its corresponding `displayName` tag value. Both fields must match to maintain consistency between the resource name and its metadata.

```bash
sed -i 's/"name": ".*"/"name": "arm-vnet-datacenter"/g' vnet-deployment-template.json

sed -i 's/"displayName": ".*"/"displayName": "arm-vnet-datacenter"/g' vnet-deployment-template.json
```

**Intent:**
- The first command targets the `name` field on the resource definition and replaces any existing value with `arm-vnet-datacenter`.
- The second command targets the `displayName` tag field and aligns it to the same value for human-readable governance visibility.

> **Edge case:** `sed` with a greedy `.*` regex applies to all matching lines. Verify the template structure does not have unintended `"name"` fields that would also be modified. Review the file after this step.

**Screenshot: VNet name and displayName tag updated via `sed`**

![Step 5 - Update VNet name and displayName tag](https://github.com/user-attachments/assets/607b23a4-a98b-4da0-b05f-815202a04276)

---

### Step 6: Update Address Space

Replace the existing address prefix with the datacenter-standard CIDR block `192.168.0.0/16`. This CIDR range provides 65,536 usable addresses, suitable for a full datacenter segment.

```bash
sed -i 's/"addressPrefixes": \[ ".*" \]/"addressPrefixes": [ "192.168.0.0\/16" ]/g' vnet-deployment-template.json
```

> **Note:** The forward slash in the CIDR notation must be escaped as `\/` within the `sed` substitution expression to prevent shell interpretation.

> **IP planning consideration:** Ensure the `192.168.0.0/16` range does not conflict with existing on-premises or peered VNet address spaces. Overlapping address spaces will prevent VNet peering and hybrid connectivity.

**Screenshot: Address space updated to `192.168.0.0/16`**

![Step 6 - Update address space](https://github.com/user-attachments/assets/7a807a9f-9b2f-4117-acda-8ec340dc6844)

---

### Step 7: Add Environment Tag

Inject an `Environment` tag into the ARM template immediately following the `displayName` tag. This tag is required for resource governance, environment classification, and cost allocation filtering.

```bash
sed -i '/"displayName": "arm-vnet-datacenter"/a \                "Environment": "KKE-datacenter"' vnet-deployment-template.json
```

**Tag added:** `"Environment": "KKE-datacenter"`

> **Governance note:** Tags applied at the resource level propagate to cost reports and compliance dashboards. Consistent tagging is a prerequisite for Azure Policy enforcement and resource group-level tagging inheritance strategies.

> **Risk:** The `sed` `/a` (append) command inserts a new line after the match. Verify indentation and comma placement in the JSON after this step, as malformed JSON will cause deployment failure.

**Screenshot: Environment tag injected into the ARM template**

![Step 7 - Add Environment tag](https://github.com/user-attachments/assets/6621f67f-185a-491a-ba08-f7c691c98a94)

---

### Step 8: Validate and Fix JSON Formatting

Before deployment, open the modified ARM template in a text editor to inspect the full JSON structure and correct any formatting issues introduced by the `sed` operations. Common issues include missing commas between tag entries and incorrect indentation.

```bash
vi vnet-deployment-template.json
```

**What to verify:**
- All tag entries are separated by commas
- No trailing commas after the last entry in any object
- The `tags` block is correctly nested inside the resource object
- `addressPrefixes` array is properly formatted
- Overall JSON structure is valid and complete

**Final validated ARM template structure:**

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "functions": [],
  "variables": {},
  "resources": [
    {
      "name": "arm-vnet-datacenter",
      "type": "Microsoft.Network/virtualNetworks",
      "apiVersion": "2023-11-01",
      "location": "[resourceGroup().location]",
      "tags": {
        "displayName": "arm-vnet-datacenter",
        "Environment": "KKE-datacenter"
      },
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "192.168.0.0/16"
          ]
        }
      }
    }
  ],
  "outputs": {}
}
```

> **Best practice:** For production environments, use `az deployment group validate` prior to `az deployment group create` to catch template schema errors without incurring resource creation.

**Screenshot: ARM template open in `vi` for visual JSON validation**

![Step 8a - Template open in vi for validation](https://github.com/user-attachments/assets/1726c5c9-e10f-464e-8bd4-3dca19a10e7a)

![Step 8b - Final validated JSON structure showing all applied modifications](https://github.com/user-attachments/assets/dca35f19-9bfa-4d10-adfe-87af52b733b5)

---

### Step 9: Deploy ARM Template

Execute the ARM template deployment using the Azure CLI. The `--mode Incremental` behavior (default) ensures only the resources defined in the template are created or updated; existing resources in the resource group that are not in the template remain untouched.

```bash
az deployment group create \
  --resource-group kml_rg_main-7dd447df1f3c4b74 \
  --template-file vnet-deployment-template.json
```

**Deployment output fields to verify:**
- `"provisioningState": "Succeeded"` confirms successful resource creation
- `outputResources` array should reference the VNet resource path under `Microsoft.Network/virtualNetworks/arm-vnet-datacenter`
- `"mode": "Incremental"` confirms safe deployment behavior
- `duration` field provides actual provisioning time (approximately `PT3.7550066S` in this run)

> **Operational note:** The deployment name defaults to the template filename (`vnet-deployment-template`). For auditability in production, supply a descriptive `--name` parameter: `--name deploy-arm-vnet-datacenter-$(date +%Y%m%d)`

**Screenshot: Deployment initiated and ARM API response received**

![Step 9a - Deployment command executed and response initiated](https://github.com/user-attachments/assets/1726c5c9-e10f-464e-8bd4-3dca19a10e7a)

![Step 9b - Deployment output showing provisioningState Succeeded and VNet resource path](https://github.com/user-attachments/assets/54ef6b46-e35b-4075-b0cc-0163ebdec926)

---

### Step 10: Verify Virtual Network Deployment

Query the deployed VNet directly to confirm the resource was created with the correct name, address space, and tags. This post-deployment verification step is essential for confirming that the template modifications were applied as intended.

```bash
az network vnet show \
  --resource-group kml_rg_main-7dd447df1f3c4b74 \
  --name arm-vnet-datacenter \
  --query '{Name:name, Address:addressSpace.addressPrefixes, Tags:tags}'
```

**Expected output:**

```json
{
  "Address": [
    "192.168.0.0/16"
  ],
  "Name": "arm-vnet-datacenter",
  "Tags": {
    "Environment": "KKE-datacenter",
    "displayName": "arm-vnet-datacenter"
  }
}
```

**Screenshot: Post-deployment verification confirming VNet name, address space, and tags**

![Step 10 - Post-deployment VNet verification](https://github.com/user-attachments/assets/2877d760-6c6a-4c45-9667-ce93da9ce880)

---

## Validation Results

| Attribute | Expected Value | Verified Value | Status |
|---|---|---|---|
| VNet Name | `arm-vnet-datacenter` | `arm-vnet-datacenter` | Passed |
| Address Space | `192.168.0.0/16` | `192.168.0.0/16` | Passed |
| Tag: displayName | `arm-vnet-datacenter` | `arm-vnet-datacenter` | Passed |
| Tag: Environment | `KKE-datacenter` | `KKE-datacenter` | Passed |
| Provisioning State | `Succeeded` | `Succeeded` | Passed |
| Deployment Mode | `Incremental` | `Incremental` | Passed |
| Region | East US | `eastus` | Passed |

---

## Key Decisions

| Decision | Rationale |
|---|---|
| Used `sed` for in-place template modification | Avoids manual editing errors; scripted and repeatable for automated pipelines |
| Address space set to `192.168.0.0/16` | Datacenter standard CIDR; provides 65,536 addresses for segment scalability |
| Used Incremental deployment mode (default) | Preserves existing resources in the resource group; safer for shared environments |
| Applied `displayName` and `Environment` tags | Supports Azure Policy compliance, cost allocation, and resource governance |
| Opened template in `vi` for manual JSON validation | Catches structural issues from `sed` operations before deployment |
| Queried VNet post-deployment with a JMESPath filter | Targeted verification confirms specific attributes without parsing full resource output |

---

## Errors and Resolutions

| Issue | Cause | Resolution |
|---|---|---|
| JSON formatting error after `sed` tag injection | `sed /a` appended the `Environment` tag line without a comma separator after `displayName` | Manually corrected comma placement in `vi` before deployment |
| `sed` regex too broad for `"name"` field | Greedy match potentially targets multiple `name` keys in complex templates | Reviewed template structure to confirm only the resource `name` field was affected; consider more targeted regex in complex templates |

---

## Best Practices

- **Always validate JSON before deployment.** Use `az deployment group validate` or a JSON linter (`python3 -m json.tool`) to catch structural errors prior to executing `az deployment group create`.
- **Use descriptive deployment names.** Supply `--name` with a timestamped, environment-specific value to make deployments queryable and auditable in the Azure portal and CLI.
- **Version-control all ARM templates.** Store templates in Git with meaningful commit messages. This enables rollback, peer review, and change history for infrastructure.
- **Verify CIDR conflicts before provisioning.** Cross-reference address spaces with existing VNets, on-premises subnets, and any planned peered networks to prevent future connectivity issues.
- **Apply tagging standards at provisioning time.** Tags are significantly harder to enforce retroactively across existing resources than they are to apply during initial deployment.
- **Prefer parameterized ARM templates for production.** Hardcoding values into templates as done here is acceptable for one-time provisioning; production templates should use `parameters` blocks for environment-specific values to support reuse across dev, staging, and production.
- **Use Incremental mode carefully in shared resource groups.** Incremental mode does not delete resources absent from the template but will update matching resources. Always confirm the resource group's existing state before deployment.

---

## Lessons Learned

- `sed` with greedy `.*` patterns requires careful review in ARM templates that contain multiple fields with the same key name, such as nested `"name"` properties. A more targeted regex or a JSON manipulation tool like `jq` is safer for complex templates.
- The `sed /a` (append after match) command does not automatically handle JSON comma placement. Any tag injection via `sed` must be followed by a manual or automated JSON validation step.
- ARM deployments via `az deployment group create` return rich JSON output including the full resource path, provider namespace, and deployment duration. Parsing this output is useful for automated post-deployment verification in CI/CD pipelines.
- ARM template deployments are idempotent by design in Incremental mode. Re-running the same deployment command after a successful run will result in a no-change update rather than resource duplication.
