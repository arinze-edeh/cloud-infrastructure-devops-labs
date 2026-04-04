# Attaching a Public IP to an Azure VM NIC via Azure CLI

> **Production-style runbook** for associating an existing Public IP address to a Virtual Machine Network Interface Card (NIC) using the Azure CLI. Covers resource discovery, cross-region considerations, troubleshooting, and validation.

---

## Table of Contents

- [Overview](#overview)
- [Problem Statement](#problem-statement)
- [Architecture Summary](#architecture-summary)
- [Prerequisites](#prerequisites)
- [Step 1: Authenticate to Azure](#step-1-authenticate-to-azure)
- [Step 2: Validate Resources and Discovery](#step-2-validate-resources-and-discovery)
- [Step 3: Troubleshoot IP Configuration Name](#step-3-troubleshoot-ip-configuration-name)
- [Step 4: Attach the Public IP to the NIC](#step-4-attach-the-public-ip-to-the-nic)
- [Step 5: Verification](#step-5-verification)
- [Validation Checklist](#validation-checklist)
- [Operational Notes](#operational-notes)
- [Lessons Learned](#lessons-learned)
- [Tags](#tags)

---

## Overview

This runbook documents the end-to-end process of attaching an existing **Azure Public IP address** to a **Virtual Machine Network Interface Card (NIC)** using the **Azure CLI**. It is intended for use by infrastructure engineers, cloud operations teams, and DevOps practitioners requiring a repeatable, auditable procedure suitable for production environments.

The document follows a **Problem > Discovery > Implementation > Validation** structure to ensure clarity and reproducibility.

---

## Problem Statement

An Azure Virtual Machine (`datacenter-vm-pip`) required a dedicated Public IP address (`datacenter-pip`) to enable direct external accessibility. The Public IP resource already existed but was not yet associated with the VM's NIC.

**Key challenges encountered:**

- The IP configuration name on the NIC did not follow the default naming convention (`ipconfig1`), causing an initial `ResourceNotFoundError`.
- The Public IP resource was provisioned in a **different region** (`westus`) from the Virtual Machine (`eastus`), requiring the use of the full resource ID for cross-region association.

---

## Architecture Summary

| Resource | Name | Details |
|---|---|---|
| **Cloud Provider** | Microsoft Azure | |
| **Resource Group** | `kml_rg_main-bf56eb2fed794029` | Region: `eastus` |
| **Virtual Machine** | `datacenter-vm-pip` | Location: `eastus` |
| **Public IP Address** | `datacenter-pip` | Location: `westus` |
| **Network Interface Card** | `datacenter-vm-pipVMNic` | Attached to VM |
| **NIC IP Config Name** | `ipconfigdatacenter-vm-pip` | Discovered via `az network nic show` |
| **Access Method** | Azure CLI | |

---

## Prerequisites

Before executing this runbook, ensure the following:

- **Azure CLI** is installed and up to date (`az --version`)
- The executing identity has the **Network Contributor** role (or equivalent) on the resource group
- The target **Public IP** and **NIC** resources already exist in the Azure subscription
- The **Public IP** is in an unassociated state (not already linked to another NIC or Load Balancer frontend)

---

## Step 1: Authenticate to Azure

Log in to the Azure CLI using the appropriate credentials. For automated pipelines, use a service principal or managed identity.

```bash
az login
```

> For non-interactive environments, use:
> ```bash
> az login --service-principal -u <APP_ID> -p <PASSWORD> --tenant <TENANT_ID>
> ```

Verify the active subscription context before proceeding:

```bash
az account show --query "{Subscription:name, SubscriptionId:id, TenantId:tenantId}" --output table
```

---

## Step 2: Validate Resources and Discovery

**Objective:** Confirm that all target resources exist, are in the expected state, and retrieve identifying metadata required for subsequent operations.

### 2a. List Resource Groups and Locations

```bash
az group list --query "[].{Name:name, Location:location}" --output table
```

**Expected output:** Confirms the resource group `kml_rg_main-bf56eb2fed794029` is present and located in `eastus`.

<img width="1030" height="696" alt="image" src="https://github.com/user-attachments/assets/561881e2-1d5b-4576-80d9-37ae515aa2ac" />

*Azure CLI output confirming the resource group name and region.*

---

### 2b. Identify the Target NIC Resource ID

```bash
az vm show \
  --name datacenter-vm-pip \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --query "networkProfile.networkInterfaces[0].id"
```

**Expected output:** The full ARM resource ID of the NIC attached to the VM, confirming `datacenter-vm-pipVMNic` is the associated interface.

<img width="1035" height="716" alt="image" src="https://github.com/user-attachments/assets/0b215c29-3c44-462e-826a-784460aad688" />

>*NIC ARM resource ID returned from the VM profile query.*

---

### 2c. Verify the Initial ip-config Update Attempt (Failed)

An initial attempt to attach the Public IP using the assumed default configuration name `ipconfig1` was made:

```bash
az network nic ip-config update \
  --name ipconfig1 \
  --nic-name datacenter-vm-pipVMNic \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --public-ip-address datacenter-pip
```

**Result:** `ResourceNotFoundError`

This failure confirmed that the IP configuration name on the NIC did not follow the default convention and required explicit discovery.

<img width="1034" height="685" alt="image" src="https://github.com/user-attachments/assets/1ee74a67-910f-4be5-bfa3-9bbfbb5a392c" />

>*ResourceNotFoundError returned when using the assumed default name `ipconfig1`.*

---

### 2d. Verify Public IP Address Details

```bash
az network public-ip list \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --output table
```

**Expected output:** Confirms `datacenter-pip` exists in `westus` with IP address `23.100.42.173` and `ProvisioningState: Succeeded`.

<img width="1033" height="558" alt="image" src="https://github.com/user-attachments/assets/e6ce3e45-5d44-48ff-9387-080733c47adc" />

>*Public IP address listing confirming resource name, region, address, and provisioning state.*

---

## Step 3: Troubleshoot IP Configuration Name

**Objective:** Identify the exact IP configuration name defined on the NIC, which is required for the update command.

The `ResourceNotFoundError` from Step 2c indicated a naming mismatch. Query the NIC directly to retrieve the actual IP configuration name:

```bash
az network nic show \
  --name datacenter-vm-pipVMNic \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --query "ipConfigurations[].name"
```

**Result:** The actual IP configuration name was identified as **`ipconfigdatacenter-vm-pip`**, not the assumed `ipconfig1`.

<img width="1034" height="681" alt="image" src="https://github.com/user-attachments/assets/68af1ea3-4f7b-4309-92b1-cede640a7fbd" />

>*Correct IP configuration name `ipconfigdatacenter-vm-pip` returned from the NIC show query.*

> **Operational Best Practice:** Never assume the IP configuration name follows a default convention. Always query the resource state using `az network nic show` prior to executing update operations.

---

## Step 4: Attach the Public IP to the NIC

**Objective:** Associate the existing Public IP `datacenter-pip` with the correct IP configuration on `datacenter-vm-pipVMNic` using the full resource ID to ensure cross-region compatibility.

### 4a. Capture the Public IP Resource ID

Store the resource ID in a shell variable to avoid manual errors and enable reuse:

```bash
PIP_ID=$(az network public-ip show \
  --name datacenter-pip \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --query id \
  -o tsv)
```

> Using the full resource ID (`$PIP_ID`) rather than just the resource name is critical when resources span different Azure regions, as it provides an unambiguous reference that bypasses regional name-resolution scope.

<img width="1034" height="795" alt="image" src="https://github.com/user-attachments/assets/1b2fe9c1-d95f-4bc6-8400-aedc332749f7" />

>*Public IP resource ID captured into the `$PIP_ID` variable for use in the update command.*

---

### 4b. Attach the Public IP to the NIC IP Configuration

Execute the NIC IP configuration update using the correct configuration name and resource ID:

```bash
az network nic ip-config update \
  --name "ipconfigdatacenter-vm-pip" \
  --nic-name datacenter-vm-pipVMNic \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --public-ip-address $PIP_ID
```

**Expected outcome:** The CLI returns a JSON payload representing the updated IP configuration object.

Key fields to confirm in the JSON response:

| Field | Expected Value |
|---|---|
| `name` | `ipconfigdatacenter-vm-pip` |
| `primary` | `true` |
| `privateIPAddress` | `10.0.0.4` |
| `provisioningState` | `Succeeded` |
| `publicIPAddress.id` | Full ARM ID of `datacenter-pip` |

<img width="1040" height="834" alt="image" src="https://github.com/user-attachments/assets/fcc044cd-7b7e-4fa6-adb9-29a8fe7047f9" />

>*JSON response from the NIC ip-config update command showing `provisioningState: Succeeded` and the linked Public IP resource.*

<img width="1040" height="834" alt="image" src="https://github.com/user-attachments/assets/fcc044cd-7b7e-4fa6-adb9-29a8fe7047f9" />

>*Full output of the successful NIC update, confirming the Public IP is now associated with the primary IP configuration.*

---

## Step 5: Verification

**Objective:** Confirm the Public IP address is correctly and actively associated with the Virtual Machine.

```bash
az vm list-ip-addresses \
  --name datacenter-vm-pip \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --output table
```

**Expected output:** The VM's public IP address should display as `23.100.42.173`, confirming successful association.

---

## Validation Checklist

| Check | Status |
|---|---|
| Public IP resource ID captured successfully into `$PIP_ID` | **PASS** |
| Correct NIC IP configuration name identified (`ipconfigdatacenter-vm-pip`) | **PASS** |
| NIC update completed with `provisioningState: Succeeded` | **PASS** |
| Public IP `23.100.42.173` confirmed linked to VM via `az vm list-ip-addresses` | **PASS** |

---

## Operational Notes

**Cross-Region Association**

The Public IP (`datacenter-pip`) was provisioned in `westus` while the Virtual Machine (`datacenter-vm-pip`) resided in `eastus`. Azure permits this association when using the full ARM resource ID. Passing the resource ID via `$PIP_ID` rather than the resource name ensures reliable cross-region resolution and avoids potential name-scoping conflicts.

**Non-Standard Resource Naming**

IP configuration names on NICs are set at provisioning time and may not follow any predictable convention. Environments provisioned via Terraform, ARM templates, or portal workflows may all generate different naming patterns. Always use `az network nic show` to enumerate actual configuration names before executing updates.

**Idempotency**

The `az network nic ip-config update` command is idempotent. Re-running it with the same parameters will not cause errors and can be safely used in automated pipelines or retry scenarios.

**Network Security Groups (NSGs)**

Attaching a Public IP makes the VM externally reachable. Ensure the associated NSG allows only the required inbound ports. Audit NSG rules after this operation using:

```bash
az network nsg list \
  --resource-group kml_rg_main-bf56eb2fed794029 \
  --output table
```

---

## Lessons Learned

**Avoid Default Assumptions**

The initial failure (`ResourceNotFoundError`) occurred because the command assumed the IP configuration name was `ipconfig1`, the Azure portal default. In environments where resources were provisioned programmatically or with custom naming conventions, this assumption will fail. Always execute discovery commands (`list`, `show`) before executing state-change commands (`update`, `delete`).

**Handling Cross-Region Resources**

Azure Public IP addresses are region-scoped resources. When a Public IP resides in a different region from its target NIC or VM, referencing the resource by name alone may be insufficient. Using the full ARM resource ID eliminates ambiguity and is the recommended pattern for any cross-region or cross-subscription operation.

**Interpreting `ResourceNotFoundError`**

A `ResourceNotFoundError` in the Azure CLI most commonly indicates a **naming mismatch** rather than a genuinely missing resource. Before concluding a resource does not exist, verify the resource name using a `list` or `show` query against the parent resource. This saves significant time in production troubleshooting.

**Scripting with Resource ID Variables**

Storing resource IDs in shell variables (e.g., `PIP_ID=$(...)`) reduces the risk of typographic errors in long ARM paths, improves command readability, and enables straightforward reuse in larger automation scripts or CI/CD pipelines. This pattern should be adopted as a standard practice in any Azure CLI scripting workflow.

---

## Tags

`azure` `networking` `vm` `public-ip` `nic` `azure-cli` `infrastructure` `devops` `cross-region` `runbook`





















# Azure VM Public IP Attachment

## Project Overview

### OBJECTIVE:

- Attach an existing Public IP to a VM NIC.

- Ensure VM is accessible via Public IP.

### Azure Environment Details
CLOUD PROVIDER:

`Microsoft Azure.`

RESOURCES:

Resource Group: `kml_rg_main-bf56eb2fed794029.`

Virtual Machine: `datacenter-vm-pip (Location: eastus).`

Public IP: `datacenter-pip (Location: westus).`

Network Interface Card (NIC): `datacenter-vm-pipVMNic.`

ACCESS METHOD:

`Azure CLI.`

## Implementation Steps

## Step 1: Authenticate
ACTION:

Login to Azure using provided credentials.

## Step 2: Validate Resources & Discovery
CONFIRM:

- List resource groups and locations: `az group list --query "[].{Name:name, Location:location}" --output table.`

- Identify Target NIC: `az vm show --name datacenter-vm-pip --resource-group kml_rg_main-bf56eb2fed794029 --query "networkProfile.networkInterfaces[0].id".`

- Verify Public IP details: `az network public-ip list --resource-group kml_rg_main-bf56eb2fed794029 --output table.`

Screenshots: `resource-validation`
<img width="1030" height="696" alt="image" src="https://github.com/user-attachments/assets/d8acd204-f1a9-43bd-b85c-6df6fc14ff53" />
<img width="1035" height="716" alt="image" src="https://github.com/user-attachments/assets/6cbf15a4-f744-433f-aee9-145e9aa1d1f8" />
<img width="1034" height="685" alt="image" src="https://github.com/user-attachments/assets/148eb405-1bf8-4fd6-99bf-778d8c06b1e2" />
<img width="1033" height="558" alt="image" src="https://github.com/user-attachments/assets/f5bdaee4-17d4-44aa-96c4-7929c8e5a2df" />

## Step 3: Troubleshoot IP Configuration Name
- COMMAND:

  -  `An initial attempt using the generic name ipconfig1 resulted in a ResourceNotFoundError.`

- Discovery: Queried the NIC to find the exact internal IP configuration name:
  -  `az network nic show --name datacenter-vm-pipVMNic --resource-group kml_rg_main-bf56eb2fed794029 --query "ipConfigurations[].name".`

- Result: Identified the correct name as `"ipconfigdatacenter-vm-pip".`

Screenshot: `nic-config-discovery`
<img width="1034" height="681" alt="image" src="https://github.com/user-attachments/assets/0d6a2286-b3ab-47a1-9adc-9e7b1048913b" />

## Step 4: Attach Public IP to NIC
COMMAND:

Capture Public IP Resource ID: `PIP_ID=$(az network public-ip show --name datacenter-pip --resource-group kml_rg_main-bf56eb2fed794029 --query id -o tsv).`

Assign datacenter-pip using the correct configuration name:

- `az network nic ip-config update \`
  -  `--name "ipconfigdatacenter-vm-pip" \`
  -  `--nic-name datacenter-vm-pipVMNic \`
  -  `--resource-group kml_rg_main-bf56eb2fed794029 \`
  -  `--public-ip-address $PIP_ID`

Screenshots: `nic-public-ip-attachment'`
<img width="1034" height="795" alt="image" src="https://github.com/user-attachments/assets/1b2fe9c1-d95f-4bc6-8400-aedc332749f7" />
<img width="1040" height="834" alt="image" src="https://github.com/user-attachments/assets/fcc044cd-7b7e-4fa6-adb9-29a8fe7047f9" />

## Step 5: Verification
CHECK:

Verify JSON response for "provisioningState": "Succeeded".

Confirm Public IP address is assigned: `az vm list-ip-addresses --name datacenter-vm-pip --resource-group kml_rg_main-bf56eb2fed794029 --output table.`

Screenshot: `public-ip-confirmed`

<img width="1040" height="834" alt="image" src="https://github.com/user-attachments/assets/fcc044cd-7b7e-4fa6-adb9-29a8fe7047f9" />

## Validation Checklist

- Public IP ID captured successfully
- NIC IP configuration name correctly identified
- NIC updated to "Succeeded" status
- Public IP 23.100.42.173 linked to VM

## Notes
- Cross-Region Association: The Public IP was located in `westus` while the VM was in `eastus`; using the resource ID `($PIP_ID)` ensured proper mapping.

- Resource Names: IP configuration names may not follow standard naming conventions; always verify with az network nic show.

## Lessons Learned

- Avoid Default Assumptions
`During the initial implementation, the command failed because it assumed the IP configuration name was ipconfig1.`

`It is critical to use az network nic show to discover the actual resource names within the environment.`

- Handling Cross-Region Resources
`The Public IP (westus) and the Virtual Machine (eastus) were located in different Azure regions.`

`Using the Resource ID instead of just the resource name provides a more robust way to link assets across different locations and avoid naming conflicts.`

- Troubleshooting "Resource Not Found" Errors
`A ResourceNotFoundError in Azure CLI is often a sign of a naming mismatch rather than a missing resource.`

`Systematic "Discovery" commands (like list and show) should always precede "Update" commands to ensure the target environment state is accurately mapped.`

- Efficient Scripting with Variables
`Storing Resource IDs in variables (e.g., $PIP_ID) makes CLI commands cleaner and significantly easier to debug or automate in larger DevOps pipelines.`

## Tags
`azure`
`networking`
`vm`
`public-ip`
`nic`
`azure-cli`
