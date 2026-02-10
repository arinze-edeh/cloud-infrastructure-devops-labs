# Azure VM Creation Using Azure CLI – Compute Lab

## Lab Overview
This lab demonstrates how to create an Azure Virtual Machine using the Azure CLI without access to the Azure Portal.  
All infrastructure provisioning is performed via command-line automation.

---

## Environment Details

| Item | Value |
|-----|------|
| Cloud Provider | Microsoft Azure |
| Service | Virtual Machines |
| Region | eastus |
| VM Name | xfusion-vm |
| OS Image | Ubuntu 22.04 |
| VM Size | Standard_B2s |
| Disk Size | 30GB |
| Storage SKU | Standard_LRS |
| Admin User | azureuser |

---

## Objective

- Create an Azure VM using CLI
- Generate SSH keys automatically
- Ensure VM is running after creation

---

## High-Level Logic (Pseudo-Code)

- LOGIN to Azure using CLI

- CREATE resource group

- CREATE virtual machine with:
  -  - Ubuntu image
  -  - Defined VM size
  -  - Admin username
  -  - SSH authentication
  -  - Standard storage
  -  - 30GB disk

VERIFY VM power state
CONFIRM VM is running
🛠️ Implementation Steps
Step 1: Azure CLI Login
📸 Screenshot Placeholder
![Azure CLI Login](screenshots/azure-login.png)

Step 2: Resource Group Creation
📸 Screenshot Placeholder
![Resource Group](screenshots/resource-group.png)

Step 3: VM Creation via Azure CLI
📸 Screenshot Placeholder
![VM Creation](screenshots/vm-create.png)

Step 4: VM Running Verification
📸 Screenshot Placeholder
![VM Running](screenshots/vm-running.png)

✅ Outcome
✔ Azure VM successfully created
✔ SSH access enabled
✔ VM running state confirmed
✔ No Azure Portal usage

📚 Key Azure Concepts Demonstrated
Azure CLI usage

Virtual machine provisioning

SSH-based authentication

Azure compute resource management

🏷️ Tags
azure vm compute azure-cli cloud-infrastructure





<img width="1031" height="688" alt="image" src="https://github.com/user-attachments/assets/d555ad61-25a9-4c24-b629-199515c68329" />
<img width="1022" height="819" alt="image" src="https://github.com/user-attachments/assets/46ef8a6a-d6f6-46ae-90b3-b551d42559bd" />
<img width="1030" height="698" alt="image" src="https://github.com/user-attachments/assets/cd85dce5-746d-4461-864d-ab3b3212b120" />
<img width="1023" height="482" alt="image" src="https://github.com/user-attachments/assets/2ca2f465-e217-4156-888d-c85acaa10fa2" />
<img width="1035" height="475" alt="image" src="https://github.com/user-attachments/assets/29aec005-52ec-41e8-8068-e6707a59c81e" />
<img width="1041" height="674" alt="image" src="https://github.com/user-attachments/assets/f61892e7-b59f-4eb2-b17d-823f3b09b2c0" />


