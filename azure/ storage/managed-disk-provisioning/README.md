# Azure Managed Disk Creation (Standard_LRS)

## Project Overview
- As part of an incremental cloud migration strategy, the Nautilus DevOps team is
migrating storage components to Microsoft Azure in controlled phases.

- This task demonstrates the creation of an Azure Managed Disk using the
Azure CLI, following DevOps and SRE best practices for automation,
repeatability, and auditability.

---

## Objective
- Provision an Azure Managed Disk with the following specifications:

- Disk Name: `nautilus-disk`
- Disk Type: `Standard_LRS`
- Disk Size: `2 GiB`
- Provisioning Method: `Azure CLI`
  
---

## Tools & Services Used
- Azure CLI
- Azure Managed Disks
- Azure Resource Groups

---

## Preconditions
- Azure CLI installed
- Authenticated Azure session (service principal)
- Existing resource group

---

## Step-by-Step Implementation

---

### Step 1: Verify Azure Account Context

- RUN `az account show`

CONFIRM:

- Subscription is active

- Authentication type = servicePrincipal

- Correct tenant and subscription in use


📸 Screenshot: `Account Details`
<img width="1032" height="618" alt="image" src="https://github.com/user-attachments/assets/42db5718-90c9-461f-a78a-2025ac7a127d" />

---

### Step 2: Identify Target Resource Group

- RUN `az group list --query "[].name" -o tsv`
- SELECT existing resource group for deployment


- **Selected Resource Group:**

`kml_rg_main-d71f6b2326b14421`


📸 Screenshot: `Resource Group List`
<img width="1028" height="621" alt="image" src="https://github.com/user-attachments/assets/d22205e4-6aff-482f-8496-9beef4330202" />

---

### Step 3: Create the Managed Disk

RUN `az disk create
--resource-group kml_rg_main-d71f6b2326b14421
--name nautilus-disk
--sku Standard_LRS
--size-gb 2`


EXPECTED RESULT:

provisioningState: `Succeeded`
diskState: `Unattached`
diskSizeGb: `2`
sku: `Standard_LRS`


📸 ScreenshotS: `Disk Creation Output`
<img width="1025" height="858" alt="image" src="https://github.com/user-attachments/assets/f1fd5a65-6a3b-4d75-aba8-28447eee72c7" />
<img width="1028" height="866" alt="image" src="https://github.com/user-attachments/assets/6b9e0617-3466-4451-801d-fe4970c01423" />
<img width="1033" height="735" alt="image" src="https://github.com/user-attachments/assets/8be0f3ad-117a-45c3-b43e-7f3093de838f" />

---

### Step 4: Verify Disk Creation

- RUN `az disk show
--resource-group kml_rg_main-d71f6b2326b14421
--name nautilus-disk
--output table`


EXPECTED OUTPUT:

| Name           | Resource Group                | Location | SKU          | Size (GiB) | Provisioning State |
|----------------|--------------------------------|----------|--------------|------------|--------------------|
| nautilus-disk  | kml_rg_main-d71f6b2326b14421   | eastus   | Standard_LRS | 2          | Succeeded          |


📸 Screenshot: `Managed Disk Verification`
<img width="1030" height="474" alt="image" src="https://github.com/user-attachments/assets/15da9d71-de80-49f1-b9dd-9d74af05467b" />

---

## Validation Checklist
- Disk name matches requirement
- Disk size is exactly 2 GiB
- Disk type is Standard_LRS
- Disk provisioning state is Succeeded
- Disk is ready for VM attachment

---

## Outcome
- Successfully provisioned an Azure Managed Disk using CLI
- Disk is unattached and ready for compute integration
- Storage component prepared for incremental migration workflow

---

## Key DevOps Takeaways
- Azure CLI enables automation-friendly infrastructure provisioning
- Service principal authentication aligns with enterprise security practices
- Incremental migration reduces operational risk
- CLI workflows are preferred for CI/CD and IaC pipelines
