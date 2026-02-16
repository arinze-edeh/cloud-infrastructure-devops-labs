# Azure NIC Attachment to Virtual Machine (CLI-Based)

## Project Overview

- TASK:
  -  Attach an existing Network Interface (NIC) to an Azure Virtual Machine using Azure CLI

- GOAL:
  -  Ensure NIC is attached successfully and VM is running

## Infrastructure Details

- Cloud Provider   : `Azure`
- Region           : `eastus`
- Virtual Machine  : `nautilus-vm`
- Network Interface: `nautilus-nic`

## Preconditions
- Azure CLI installed and authenticated
- VM exists in target resource group
- NIC exists in same resource group
- VM must be deallocated before NIC attachment

## Step-by-Step Implementation (Azure CLI)

## Step 1: Identify Resource Group
- COMMAND:
    `az group list --query "[].name" -o tsv`

- OUTPUT:
    `kml_rg_main-ae708aa3b65d4be6`

screenshot: `az-group-list`
<img width="1032" height="692" alt="image" src="https://github.com/user-attachments/assets/e72f0ede-4262-4b54-915f-2b9d59b27314" />

## Step 2: Deallocate the Virtual Machine
- REASON:
  -  Azure requires VM to be deallocated before NIC changes

- COMMAND:
  -  `az vm deallocate \`
    -  `--resource-group kml_rg_main-ae708aa3b65d4be6 \`
    -  `--name nautilus-vm`

screenshot: `vm-deallocated`
<img width="1031" height="781" alt="image" src="https://github.com/user-attachments/assets/ab6d4de4-31c7-450e-b110-1ae142658f99" />

## Step 3: Attach Network Interface to VM
- COMMAND:
  -  `az vm nic add \`
    -  `--resource-group kml_rg_main-ae708aa3b65d4be6 \`
    -  `--vm-name nautilus-vm \`
    -  `--nics nautilus-nic`

- EXPECTED OUTPUT:
    - Existing primary NIC retained
    - nautilus-nic attached as secondary NIC

screenshot: `nic-attached-cli-output`
<img width="1031" height="693" alt="image" src="https://github.com/user-attachments/assets/ac4e0c9e-215f-44df-bf03-95f118088ebf" />

## Step 4: Start the Virtual Machine
- COMMAND:
  -  `az vm start \`
    -  `--resource-group kml_rg_main-ae708aa3b65d4be6 \`
    -  `--name nautilus-vm`

screenshot: `vm-started`
<img width="1036" height="671" alt="image" src="https://github.com/user-attachments/assets/f5602539-adee-4844-acd3-c3632b26bbbf" />

## Step 5: Verify VM Power State
- COMMAND:
  -  `az vm show -d \`
    -  `--resource-group kml_rg_main-ae708aa3b65d4be6 \`
    -  `--name nautilus-vm \`
    -  `--query "powerState"`

- EXPECTED OUTPUT:
    `"VM running"`

screenshot: `vm-running-status`
<img width="1036" height="683" alt="image" src="https://github.com/user-attachments/assets/3b53d975-7360-4bef-add4-218ca705fc83" />

## Validation Checklist
- VM state == `running`
- NIC `nautilus-nic` attached
- No CLI errors returned

## Outcome
- RESULT:
  -  Existing network interface successfully attached to Azure VM using Azure CLI
- Notes
  -  VM must be deallocated before NIC modification
  -  Primary NIC remains unchanged
  -  Secondary NIC added successfully

## Tags
`azure`
`azure-cli`
`networking`
`virtual-machines`
`nic`
`cloud-operations`
