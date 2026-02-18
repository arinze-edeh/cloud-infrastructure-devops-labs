# Azure VM Resize Using Azure CLI

- PROJECT TYPE: `Azure Compute`  
- SERVICE: `Virtual Machines`  
- OPERATION: `VM Resize (SKU Change)`  
- REGION: `centralus`  

---

## Project Overview

- This project demonstrates how to resize an existing Azure Virtual Machine
using the Azure CLI in order to optimize resource usage.

- A running VM named `"xfusion-vm"` was identified as underutilized.
- The VM was resized from a smaller SKU to a larger SKU while ensuring
that the VM remains in a running state after the operation.

This task validates hands-on skills in:
- Azure CLI usage
- VM lifecycle management
- Compute resource optimization
- State verification after configuration change

---

## Project Objectives

- Identify an existing Azure Virtual Machine
- Verify the current VM size
- Resize the VM to a new compute SKU
- Ensure the VM is running after resize
- Confirm the final VM size and state

---

## Azure Scope and Constraints

- SUBSCRIPTION: `Azure Free Lab Environment`
- RESOURCE GROUP: `KML_RG_MAIN-9642FE6E452040F1`  
- VIRTUAL MACHINE NAME: `xfusion-vm`  
- LOCATION: `centralus`  

---

## Prerequisites

- Azure CLI installed and configured
- Active Azure lab credentials
- Permission to manage Virtual Machines
- VM must exist in the target resource group

---

## Step-by-Step Implementation

### Step 1: Retrieve Azure Credentials

ACTION:
- Display Azure lab credentials

COMMAND:
    RUN `showcreds`

EXPECTED OUTPUT:
- Azure portal URL
- Username
- Password
- Session expiry time

SCREENSHOT:
  <img width="1034" height="659" alt="image" src="https://github.com/user-attachments/assets/7355fc86-7d19-40c3-9455-d30adda8155a" />

---

### Step 2: List Available Virtual Machines

ACTION:
- List all VMs in the subscription
- Display VM name, resource group, and location

COMMAND:
    `RUN az vm list
        FILTER output to:
            VM_Name
            ResourceGroup
            Location
        FORMAT output as table`

EXPECTED RESULT:
- VM `"xfusion-vm"` found
- Resource group confirmed
- Location confirmed as centralus

SCREENSHOT:
   <img width="1029" height="555" alt="image" src="https://github.com/user-attachments/assets/09466e95-8808-4ab4-bc4e-3a2f9ca970b0" />

---

### Step 3: Verify Current VM Size

ACTION:
- Check the current hardware size of the VM

COMMAND:
    `RUN az vm show
        --resource-group KML_RG_MAIN-9642FE6E452040F1
        --name xfusion-vm
        QUERY hardwareProfile.vmSize`

EXPECTED RESULT:
    Standard_B1s

SCREENSHOT:
    [Screenshot: VM size before resize]

---

### Step 4: Resize the Virtual Machine

ACTION:
- Update the VM hardware profile to a larger SKU

COMMAND:
    `RUN az vm update
        --resource-group KML_RG_MAIN-9642FE6E452040F1
        --name xfusion-vm
        --size Standard_B2s`

SYSTEM NOTE:
- `"--size"` parameter is currently in preview
- Operation updates VM hardware profile successfully

EXPECTED RESULT:
- VM size changed to Standard_B2s
- Provisioning state shows "Succeeded"

SCREENSHOT:
    [Screenshot: VM resize command execution]

---

### Step 5: Start the Virtual Machine

ACTION:
- Ensure the VM is powered on after resize

COMMAND:
    `RUN az vm start
        --resource-group KML_RG_MAIN-9642FE6E452040F1
        --name xfusion-vm`

EXPECTED RESULT:
- VM transitions to running state

SCREENSHOT:
    [Screenshot: VM start command]

---

### Step 6: Verify Final VM Size and State

ACTION:
- Confirm the VM size and running status

COMMAND:
    `RUN az vm get-instance-view
        --resource-group KML_RG_MAIN-9642FE6E452040F1
        --name xfusion-vm
        QUERY:
            hardwareProfile.vmSize
            instanceView.statuses[1].displayStatus`

EXPECTED RESULT:
    Size  = Standard_B2s
    State = VM running

SCREENSHOT:
    <img width="1031" height="870" alt="image" src="https://github.com/user-attachments/assets/90502930-4c46-4749-9edb-f7f20fc9ef99" />

---

## Validation Checklist

- VM exists in centralus region
- Original VM size verified
- VM resized successfully
- VM is running after resize
- Final size confirmed as Standard_B2s

---

## Outcome

- The Virtual Machine `"xfusion-vm"` was successfully resized from
`Standard_B1s` to `Standard_B2s` using `Azure CLI`.

- The VM remained operational and was verified to be running
after the resize operation, meeting all project requirements.




<img width="1028" height="577" alt="image" src="https://github.com/user-attachments/assets/a67aa12e-9d6a-44a2-826a-66287742fb54" />
<img width="1037" height="825" alt="image" src="https://github.com/user-attachments/assets/3115eb1a-2257-40d4-a724-86617c7551e7" />
<img width="1035" height="826" alt="image" src="https://github.com/user-attachments/assets/93424d0b-a961-487a-b292-554a03f0bcb2" />
<img width="1037" height="829" alt="image" src="https://github.com/user-attachments/assets/9e8c714c-12d8-48e8-a0b5-6c623a16f935" />
<img width="1037" height="650" alt="image" src="https://github.com/user-attachments/assets/d5abc90e-4c9a-4575-a1ca-f1545543fae4" />
<img width="1031" height="867" alt="image" src="https://github.com/user-attachments/assets/38a6cbf7-df18-4384-89e8-8da11e630de8" />


