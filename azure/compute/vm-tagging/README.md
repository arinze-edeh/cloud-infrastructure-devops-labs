# Azure VM Tagging Using Azure CLI

PROJECT TYPE:
    Azure Compute – Virtual Machine Management

TASK:
    Apply an Environment tag to an existing Azure Virtual Machine
    using Azure CLI

VM NAME:
    devops-vm

TAG APPLIED:
    Environment=dev

SUBSCRIPTION:
    Azure Free Labs

REGION:
    southcentralus

---

## Project Overview

This project demonstrates how to use Azure CLI
to apply metadata tags to an existing Virtual Machine.

Tagging helps with:
    - Resource organization
    - Environment identification
    - Cost management
    - Governance and automation

The task was completed without stopping or restarting the VM.

---

## Problem Statement

During infrastructure migration,
a virtual machine was discovered to be missing
the required environment tag.

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
    Confirm active Azure subscription and login context

COMMAND:
    `az account show`

EXPECTED OUTPUT:
    - Subscription name
    - Subscription ID
    - Tenant ID
    - Account state = Enabled

SCREENSHOT: `az account show output`

---

### Step 2: List Available Subscriptions

ACTION:
    Verify correct subscription is set as default

COMMAND:
    `az account list --output table`

EXPECTED RESULT:
    - Azure Free Labs subscription
    - IsDefault = True

SCREENSHOT: `az account list output`

---

### Step 3: Identify Resource Group of the VM

ACTION:
    Query Azure to find the resource group
    containing the target VM

COMMAND:
    `az vm list --query "[?name=='devops-vm'].{ResourceGroup:resourceGroup}"`

EXPECTED RESULT:
    ResourceGroup = `KML_RG_MAIN-C849AEA3729A4D94`

SCREENSHOT: `VM resource group query output`

---

### Step 4: Apply Environment Tag to the VM

ACTION:
    Update the VM metadata to include Environment tag

COMMAND:
    az vm update \
      --resource-group KML_RG_MAIN-C849AEA3729A4D94 \
      --name devops-vm \
      --set tags.Environment=dev

EXPECTED RESULT:
    - ProvisioningState = Succeeded
    - Tag added successfully
    - VM remains running

SCREENSHOT: `az vm update success output`

---

### Step 5: Verify Tag Application

ACTION:
    Retrieve VM tags to confirm update

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

The virtual machine "devops-vm"
was successfully tagged using Azure CLI.

Tag Applied:
    `Environment=dev`

This ensures improved governance,
resource visibility,
and alignment with DevOps best practices.







<img width="1027" height="849" alt="image" src="https://github.com/user-attachments/assets/79ce7653-1946-4a3d-9b85-b3adbffdc4db" />
<img width="1029" height="847" alt="image" src="https://github.com/user-attachments/assets/c4144f14-6073-4529-b12a-d49e7fe87403" />
<img width="1027" height="790" alt="image" src="https://github.com/user-attachments/assets/be960db0-9bf1-4250-9ce0-af28fb4e0fb1" />
<img width="1035" height="859" alt="image" src="https://github.com/user-attachments/assets/f27a2e76-65fa-46bd-b9b2-dd2199327d5b" />
<img width="1017" height="860" alt="image" src="https://github.com/user-attachments/assets/110218dc-3a5e-4ff2-a9de-674851beb824" />
<img width="1028" height="862" alt="image" src="https://github.com/user-attachments/assets/bfb9b7c6-4480-41a0-8a21-7f5ec658cdda" />
<img width="1028" height="849" alt="image" src="https://github.com/user-attachments/assets/364409ed-51ac-4d51-a744-458ba3e81428" />
<img width="1036" height="446" alt="image" src="https://github.com/user-attachments/assets/776cd497-40aa-4a77-aeda-d54ff0728790" />



