# Project: Attach Existing Managed Disk to Azure Virtual Machine


## Objective
- Attach an existing Azure managed disk to an existing virtual machine
- Ensure the disk is attached as a **data disk**
- Validate that the virtual machine initialization is complete

---

## Environment Details
- Cloud Provider: `Azure`
- Region: `eastus`
- Virtual Machine Name: `devops-vm`
- Managed Disk Name: `devops-disk`

---

## Prerequisites
- Azure Portal access
- Existing virtual machine
- Existing managed disk
- VM and disk must be in the same region
- VM must be initialized before submission

---

## Step-by-Step Implementation

---

### Step 1: Login to Azure Portal
- OPEN Azure Portal using Portal URL
- ENTER username
- ENTER password
- CONFIRM successful login

screenshot: `azure-portal-login`

<img width="1828" height="946" alt="image" src="https://github.com/user-attachments/assets/3830c7b2-363a-47e4-bea4-c5c816c3ff06" />

---

### Step 2: Verify Virtual Machine
- NAVIGATE to `Virtual Machines`
- SELECT `devops-vm`
- VERIFY:
  - VM exists
  - Region = eastus
  - Status = Running or Initialized

screenshot: `vm-overview`
<img width="1830" height="950" alt="image" src="https://github.com/user-attachments/assets/05e3d5ad-6c1a-43ca-82e9-a1f6f8318c82" />

---

### Step 3: Verify Managed Disk
- NAVIGATE to `Disks`
- SELECT `devops-disk`
- VERIFY:
  - Disk exists
  - Region = eastus
  - Disk is not attached to any VM

screenshot: `disk-overview`
<img width="1855" height="946" alt="image" src="https://github.com/user-attachments/assets/26a4cc7f-79ca-4314-9a20-de66e74760f5" />

---

### Step 4: Attach Managed Disk to VM
- OPEN virtual machine `devops-vm`
- NAVIGATE to `Settings → Disks`
- CLICK `Attach existing disk`
- SELECT disk `devops-disk`
- SET disk type = Data disk
- CLICK `Save`

screenshot: `attach-disk-configuration`
<img width="1919" height="949" alt="image" src="https://github.com/user-attachments/assets/a929ec8e-dfcf-48ba-bd53-9e3d3f740366" />
<img width="1869" height="949" alt="image" src="https://github.com/user-attachments/assets/ebb5dec2-5d87-493c-9aac-e89486f5c6dc" />

---

### Step 5: Validate Disk Attachment
- WAIT for deployment to complete
- CONFIRM:
  - Disk appears under Data disks
  - Disk status = Attached

screenshot: `disk-attached-success`
<img width="1822" height="952" alt="image" src="https://github.com/user-attachments/assets/c177604c-3614-4571-8452-45bb3d19cc8a" />

---

### Validation Checklist
- VM `devops-vm` is running
- Disk `devops-disk` is attached as a data disk
- No deployment or configuration errors
- VM initialization completed

---

### Tags
`azure` `compute`
`virtual-machine`
`managed-disk`
`storage`
`devops`
`cloud-operations`
